# WHMCS Gateway From Scratch Workflow

## Description
Create a custom payment gateway module for WHMCS from scratch.

## Prerequisites
- WHMCS installation
- Payment gateway API documentation
- PHP 8.1+
- SSL certificate

## Module Structure
```
modules/gateways/
└── payment/
    └── clicodes_gateway/
        ├── clicodes_gateway.php      # Main gateway file
        ├── callback.php               # IPN/webhook handler
        ├── logo.png                   # Gateway logo
        ├── LICENSE
        └── README.md
```

## Steps

### Step 1: Create Gateway Directory
```bash
mkdir -p /var/www/whmcs/modules/gateways/clicodes_gateway
cd /var/www/whmcs/modules/gateways/clicodes_gateway
```

### Step 2: Create Main Gateway File
```php
<?php
/**
 * WHMCS Payment Gateway - CLICodes Gateway
 *
 * @copyright Copyright (c) 2024 Your Company
 * @license https://example.com/license
 * @version 1.0.0
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Define gateway configuration
 */
function clicodes_gateway_MetaData()
{
    return [
        'DisplayName' => 'CLICodes Payment Gateway',
        'APIVersion' => '1.0',
        'DisableLocalCreditCardInput' => false,
        'TokenisedStorageAllowed' => false,
    ];
}

/**
 * Define configuration options
 */
function clicodes_gateway_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'CLICodes Payment',
        ],
        'environment' => [
            'Type' => 'dropdown',
            'Label' => 'Environment',
            'Options' => [
                'sandbox' => 'Sandbox',
                'production' => 'Production',
            ],
            'Default' => 'sandbox',
        ],
        'merchantId' => [
            'Type' => 'text',
            'Label' => 'Merchant ID',
            'Size' => '35',
            'Description' => 'Your merchant account ID',
        ],
        'apiKey' => [
            'Type' => 'text',
            'Label' => 'API Key',
            'Size' => '64',
            'Description' => 'Your API key',
        ],
        'apiSecret' => [
            'Type' => 'password',
            'Label' => 'API Secret',
            'Size' => '64',
            'Description' => 'Your API secret',
        ],
        'webhookSecret' => [
            'Type' => 'password',
            'Label' => 'Webhook Secret',
            'Size' => '64',
            'Description' => 'Secret for verifying webhook signatures',
        ],
        'testUsername' => [
            'Type' => 'text',
            'Label' => 'Test Username',
            'Size' => '35',
        ],
        'testPassword' => [
            'Type' => 'password',
            'Label' => 'Test Password',
            'Size' => '35',
        ],
        'settlementAccount' => [
            'Type' => 'text',
            'Label' => 'Settlement Account',
            'Description' => 'Bank account for settlements',
        ],
        'additionalFee' => [
            'Type' => 'text',
            'Label' => 'Additional Fee (%)',
            'Description' => 'Additional percentage fee to add to transactions',
        ],
    ];
}

/**
 * Link to payment form
 */
function clicodes_gateway_link($params)
{
    // Client information
    $invoiceId = $params['invoiceid'];
    $description = $params["description"];
    $amount = $params['amount'];
    $currency = $params['currency'];
    
    // Client details
    $clientFirstName = $params['clientdetails']['firstname'];
    $clientLastName = $params['clientdetails']['lastname'];
    $clientEmail = $params['clientdetails']['email'];
    
    // System details
    $systemUrl = rtrim($params['systemurl'], '/');
    $returnUrl = $params['returnurl'];
    $merchantId = $params['merchantId'];
    $environment = $params['environment'];
    
    // Build API endpoint
    $apiUrl = ($environment === 'sandbox')
        ? 'https://sandbox.gateway.example.com/v1/checkout'
        : 'https://gateway.example.com/v1/checkout';
    
    // Generate transaction reference
    $transactionRef = 'INV' . $invoiceId . '_' . time();
    
    // Store transaction reference in session for verification
    $_SESSION['clicodes_gateway_ref'] = $transactionRef;
    
    // Build form
    $html = '<form id="clicodes_payment_form" action="' . $apiUrl . '" method="POST">';
    $html .= '<input type="hidden" name="merchant_id" value="' . htmlspecialchars($merchantId) . '">';
    $html .= '<input type="hidden" name="order_id" value="' . $transactionRef . '">';
    $html .= '<input type="hidden" name="amount" value="' . number_format($amount, 2, '.', '') . '">';
    $html .= '<input type="hidden" name="currency" value="' . $currency . '">';
    $html .= '<input type="hidden" name="description" value="' . htmlspecialchars($description) . '">';
    $html .= '<input type="hidden" name="customer_name" value="' . htmlspecialchars($clientFirstName . ' ' . $clientLastName) . '">';
    $html .= '<input type="hidden" name="customer_email" value="' . htmlspecialchars($clientEmail) . '">';
    $html .= '<input type="hidden" name="return_url" value="' . htmlspecialchars($returnUrl) . '">';
    $html .= '<input type="hidden" name="cancel_url" value="' . htmlspecialchars($params['returnurl'] . '?cancel=true') . '">';
    $html .= '<input type="hidden" name="notify_url" value="' . $systemUrl . '/modules/gateways/clicodes_gateway/callback.php">';
    $html .= '<input type="hidden" name="signature" value="' . clicodes_generate_signature($params, $amount, $transactionRef) . '">';
    
    $html .= '<div class="payment-button-container">';
    $html .= '<button type="submit" class="btn btn-primary btn-lg">';
    $html .= '<i class="fa fa-credit-card"></i> Pay with CLICodes';
    $html .= '</button>';
    $html .= '</div>';
    $html .= '</form>';
    
    // Add JavaScript for auto-submit
    $html .= '<script type="text/javascript">';
    $html .= 'document.addEventListener("DOMContentLoaded", function() {';
    $html .= '    // Optional: Auto-submit after page load';
    $html .= '    // document.getElementById("clicodes_payment_form").submit();';
    $html .= '});';
    $html .= '</script>';
    
    return $html;
}

/**
 * Generate signature for API request
 */
function clicodes_generate_signature($params, $amount, $transactionRef)
{
    $data = $params['merchantId'] . '|';
    $data .= $transactionRef . '|';
    $data .= number_format($amount, 2, '.', '') . '|';
    $data .= $params['environment'] === 'sandbox' ? 'sandbox' : 'live';
    
    return hash_hmac('sha256', $data, $params['apiSecret']);
}

/**
 * Refund transaction
 */
function clicodes_gateway_refund($params)
{
    $transactionId = $params['transid'];
    $amount = $params['amount'];
    $environment = $params['environment'];
    
    $apiUrl = ($environment === 'sandbox')
        ? 'https://sandbox.gateway.example.com/v1/refund'
        : 'https://gateway.example.com/v1/refund';
    
    $data = [
        'transaction_id' => $transactionId,
        'amount' => $amount,
        'reason' => 'Customer requested refund via WHMCS',
    ];
    
    $response = clicodes_api_call($apiUrl, $data, $params);
    
    if ($response['success']) {
        return [
            'status' => 'success',
            'rawdata' => $response,
            'transid' => $response['refund_id'],
            'refundid' => $response['refund_id'],
        ];
    }
    
    return [
        'status' => 'error',
        'rawdata' => $response,
        'failedreason' => $response['error_message'] ?? 'Refund failed',
    ];
}

/**
 * API call helper
 */
function clicodes_api_call($url, $data, $params)
{
    $headers = [
        'Content-Type: application/json',
        'Authorization: Bearer ' . $params['apiKey'],
    ];
    
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => $url,
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode($data),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => $headers,
        CURLOPT_TIMEOUT => 30,
        CURLOPT_SSL_VERIFYPEER => true,
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    $error = curl_error($ch);
    curl_close($ch);
    
    if ($error) {
        return [
            'success' => false,
            'error' => $error,
        ];
    }
    
    $result = json_decode($response, true);
    
    return [
        'success' => ($httpCode === 200 || $httpCode === 201),
        'http_code' => $httpCode,
        'data' => $result,
    ];
}
```

### Step 3: Create Callback Handler
```php
<?php
/**
 * CLICodes Gateway Callback Handler
 * Processes payment notifications and IPN
 */

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../../../includes/gatewayfunctions.php';
require_once __DIR__ . '/../../../modules/gateways/clicodes_gateway.php';

// Get gateway parameters
$gatewayParams = getGatewayVariables('clicodes_gateway');

if (!$gatewayParams['type']) {
    die("Gateway Not Active");
}

// Get raw input
$rawInput = file_get_contents('php://input');
$input = json_decode($rawInput, true);

// Verify signature
$signature = $_SERVER['HTTP_X_SIGNATURE'] ?? '';
$expectedSignature = hash_hmac('sha256', $rawInput, $gatewayParams['webhookSecret']);

if (!hash_equals($expectedSignature, $signature)) {
    logTransaction($gatewayParams['paymentmethod'], $input, 'Invalid Signature');
    http_response_code(403);
    die('Invalid signature');
}

// Process based on event type
$event = $input['event'] ?? '';

switch ($event) {
    case 'payment.completed':
        handlePaymentComplete($input, $gatewayParams);
        break;
        
    case 'payment.failed':
        handlePaymentFailed($input, $gatewayParams);
        break;
        
    case 'refund.processed':
        handleRefundProcessed($input, $gatewayParams);
        break;
        
    default:
        logTransaction($gatewayParams['paymentmethod'], $input, 'Unknown Event');
}

// Send 200 OK
http_response_code(200);
die('OK');

/**
 * Handle successful payment
 */
function handlePaymentComplete($input, $gatewayParams)
{
    $transactionId = $input['transaction_id'] ?? '';
    $invoiceId = $input['order_id'] ?? '';
    $amount = $input['amount'] ?? 0;
    $currency = $input['currency'] ?? 'USD';
    
    // Extract invoice ID from order reference
    preg_match('/INV(\d+)_/', $invoiceId, $matches);
    $invoiceId = $matches[1] ?? 0;
    
    // Check if already processed
    $existing = select_query('tblgatewaylog', 'id', [
        'gateway' => 'clicodes_gateway',
        'data' => ['LIKE', '%' . $transactionId . '%']
    ]);
    
    if (mysql_num_rows($existing) > 0) {
        logTransaction($gatewayParams['paymentmethod'], $input, 'Duplicate Transaction');
        return;
    }
    
    // Add invoice payment
    $result = addInvoicePayment(
        $invoiceId,
        $transactionId,
        $amount,
        0,
        'clicodes_gateway'
    );
    
    if ($result) {
        logTransaction($gatewayParams['paymentmethod'], $input, 'Success');
    } else {
        logTransaction($gatewayParams['paymentmethod'], $input, 'Failed');
    }
}

/**
 * Handle failed payment
 */
function handlePaymentFailed($input, $gatewayParams)
{
    $reason = $input['error_message'] ?? 'Payment failed';
    logTransaction($gatewayParams['paymentmethod'], $input, 'Failed: ' . $reason);
}

/**
 * Handle refund
 */
function handleRefundProcessed($input, $gatewayParams)
{
    $transactionId = $input['original_transaction_id'] ?? '';
    $refundId = $input['refund_id'] ?? '';
    $amount = $input['amount'] ?? 0;
    
    logTransaction($gatewayParams['paymentmethod'], $input, 'Refund Processed');
}
```

### Step 4: Create Gateway Logo
```bash
# Create 150x50 pixel logo.png
# Use PNG format with transparent background
# Place in gateway directory
```

### Step 5: Install and Test
```bash
# Copy gateway to WHMCS
cp -r clicodes_gateway /var/www/whmcs/modules/gateways/

# Set permissions
chown -R www-data:www-data /var/www/whmcs/modules/gateways/clicodes_gateway
chmod 644 /var/www/whmcs/modules/gateways/clicodes_gateway/*.php

# Configure in WHMCS Admin
# Go to: Configuration > Payment Gateways
# Activate CLICodes Gateway
# Configure API credentials
```

### Step 6: Test Sandbox
```php
// Test with sandbox credentials
// Verify webhook endpoint is reachable
// Test complete payment flow
// Test refund functionality
```

### Step 7: Production Deployment
```bash
# 1. Update to production credentials
# 2. Enable production environment
# 3. Test with small amount
# 4. Monitor transactions
# 5. Enable monitoring alerts
```

## Gateway Requirements Checklist
- [ ] Support for one-time payments
- [ ] Support for refunds
- [ ] Webhook/IPN handling
- [ ] Signature verification
- [ ] Error handling
- [ ] Logging
- [ ] Sandbox environment
- [ ] SSL required
- [ ] PCI compliance considerations

## Tags
- payment-gateway
- module-development
- development
- payment-processing