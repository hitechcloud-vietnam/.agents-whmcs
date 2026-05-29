# WHMCS Gateway Module Development

## Overview

This workflow guides you through developing a payment gateway module for WHMCS. Gateway modules enable processing payments through various payment providers like Stripe, PayPal, or custom solutions.

## Prerequisites

- WHMCS v8.0+
- PHP 7.4+
- Payment provider API documentation
- SSL certificate for production
- Test merchant account from provider

## Step-by-Step Instructions

### Step 1: Create Module Directory Structure

Create your gateway module at `/modules/gateways/yourgateway/`:

```
/modules/gateways/
    ├── yourgateway/
    │   ├── yourgateway.php         # Main gateway file
    │   ├── callback.php            # IPN/callback handler
    │   ├── refund.php              # Refund handler (optional)
    │   └── lang/
    │       └── english.php
    └── callback/
        └── yourgateway.php         # Legacy callback
```

### Step 2: Create the Main Gateway File

Create `yourgateway.php`:

```php
<?php
/**
 * WHMCS Gateway Module - YourGateway
 *
 * @copyright Copyright (c) 2024 Your Name
 * @license https://whmcs.com/legal/
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Gateway module configuration.
 *
 * @return array Configuration options
 */
function YourGateway_MetaData()
{
    return [
        'DisplayName' => 'Your Gateway Name',
        'APIVersion' => '1.1',
        'Deprecated' => false,
        'GatewayType' => 'Credit Card, Bank Transfer',
        'Insurance' => 'Provides merchant protection',
        'Features' => [
            'Products' => true,
            'Services' => true,
            'AddFunds' => true,
            'AffiliatePayouts' => false,
            'OneTime' => true,
            'Recurring' => true,
            'CardStorage' => true,
        ],
    ];
}

/**
 * Gateway configuration fields.
 *
 * @return array Configuration fields
 */
function YourGateway_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Gateway Name',
        ],
        'apiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '40',
            'Description' => 'Your gateway API key',
        ],
        'apiSecret' => [
            'FriendlyName' => 'API Secret',
            'Type' => 'password',
            'Size' => '40',
            'Description' => 'Your gateway API secret',
        ],
        'merchantId' => [
            'FriendlyName' => 'Merchant ID',
            'Type' => 'text',
            'Size' => '20',
        ],
        'environment' => [
            'FriendlyName' => 'Environment',
            'Type' => 'dropdown',
            'Options' => [
                'live' => 'Live',
                'test' => 'Sandbox/Test',
            ],
            'Default' => 'test',
        ],
        'webhookSecret' => [
            'FriendlyName' => 'Webhook Secret',
            'Type' => 'password',
            'Size' => '40',
            'Description' => 'For verifying webhook signatures',
        ],
        'allowPartialRefund' => [
            'FriendlyName' => 'Allow Partial Refunds',
            'Type' => 'yesno',
            'Description' => 'Allow refunds less than original amount',
            'Default' => 'yes',
        ],
    ];
}

/**
 * Link to gateway for payment.
 *
 * @param array $params Payment parameters
 * @return array Redirect data
 */
function YourGateway_link(array $params)
{
    $environment = $params['environment'] ?? 'test';
    $apiEndpoint = ($environment === 'live')
        ? 'https://api.yourgateway.com/v1'
        : 'https://sandbox.yourgateway.com/v1';

    $transactionId = uniqid('whmcs_');

    // Store transaction ID for callback matching
    $params['merchantId']; // Use this to store/ref

    $returnUrl = rtrim($params['systemurl'], '/') . '/modules/gateways/callback/yourgateway.php';

    $htmlOutput = '<form id="yourgateway_payment_form" method="POST" action="' . $apiEndpoint . '/checkout">
        <input type="hidden" name="merchant_id" value="' . htmlspecialchars($params['merchantId']) . '">
        <input type="hidden" name="amount" value="' . $params['amount'] . '">
        <input type="hidden" name="currency" value="' . $params['currency'] . '">
        <input type="hidden" name="order_id" value="' . $params['invoiceid'] . '">
        <input type="hidden" name="transaction_id" value="' . $transactionId . '">
        <input type="hidden" name="return_url" value="' . $returnUrl . '?id=' . $transactionId . '">
        <input type="hidden" name="cancel_url" value="' . $params['returnurl'] . '">
        <input type="hidden" name="customer_email" value="' . htmlspecialchars($params['clientdetails']['email']) . '">
        <input type="hidden" name="customer_name" value="' . htmlspecialchars($params['clientdetails']['firstname'] . ' ' . $params['clientdetails']['lastname']) . '">
        <input type="hidden" name="description" value="Invoice #' . $params['invoiceid'] . '">
        <input type="hidden" name="signature" value="' . htmlspecialchars(YourGateway_generateSignature($params, $transactionId)) . '">
        <div class="text-center">
            <button type="submit" class="btn btn-primary btn-lg">
                Pay Now with Your Gateway
            </button>
        </div>
    </form>';

    return [
        'type' => 'form',
        'html' => $htmlOutput,
    ];
}

/**
 * Generate signature for payment request.
 *
 * @param array $params
 * @param string $transactionId
 * @return string
 */
function YourGateway_generateSignature(array $params, string $transactionId): string
{
    $data = $params['merchantId'] . '|'
        . $params['invoiceid'] . '|'
        . $params['amount'] . '|'
        . $params['currency'] . '|'
        . $transactionId;

    return hash_hmac('sha256', $data, $params['apiSecret']);
}

/**
 * Refund transaction.
 *
 * @param array $params Refund parameters
 * @return array Refund result
 */
function YourGateway_refund(array $params)
{
    try {
        $environment = $params['environment'] ?? 'test';
        $apiEndpoint = ($environment === 'live')
            ? 'https://api.yourgateway.com/v1'
            : 'https://sandbox.yourgateway.com/v1';

        $ch = curl_init($apiEndpoint . '/refunds');

        $postData = [
            'transaction_id' => $params['transid'],
            'amount' => $params['amount'],
            'reason' => $params['reason'] ?? 'Customer request',
        ];

        $headers = [
            'Authorization: Bearer ' . $params['apiKey'],
            'Content-Type: application/json',
        ];

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($postData),
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $result = json_decode($response, true);

        if ($httpCode !== 200 && $httpCode !== 201) {
            return [
                'status' => 'error',
                'rawdata' => $result,
                'message' => $result['message'] ?? 'Refund failed',
            ];
        }

        return [
            'status' => 'success',
            'refundid' => $result['refund_id'],
            'rawdata' => $result,
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'message' => $e->getMessage(),
        ];
    }
}

/**
 * Storing credit card.
 *
 * @param array $params
 * @return array
 */
function YourGateway_storeremote(array $params)
{
    // Tokenization implementation
    try {
        $environment = $params['environment'] ?? 'test';
        $apiEndpoint = ($environment === 'live')
            ? 'https://api.yourgateway.com/v1'
            : 'https://sandbox.yourgateway.com/v1';

        $ch = curl_init($apiEndpoint . '/tokens');

        $postData = [
            'card' => [
                'number' => $params['cardnum'],
                'exp_month' => $params['cardexp'],
                'exp_year' => $params['cardexp2'],
                'cvv' => $params['cardcvv'],
            ],
            'customer' => [
                'email' => $params['clientdetails']['email'],
                'name' => $params['clientdetails']['firstname'] . ' ' . $params['clientdetails']['lastname'],
            ],
        ];

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($postData),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $params['apiKey'],
                'Content-Type: application/json',
            ],
            CURLOPT_RETURNTRANSFER => true,
        ]);

        $response = curl_exec($ch);
        $result = json_decode($response, true);
        curl_close($ch);

        if (isset($result['token'])) {
            return [
                'success' => true,
                'token' => $result['token'],
            ];
        }

        return [
            'success' => false,
            'error' => $result['message'] ?? 'Tokenization failed',
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Capture authorized payment.
 *
 * @param array $params
 * @return array
 */
function YourGateway_capture(array $params)
{
    // For pre-auth workflows
    try {
        $apiEndpoint = ($params['environment'] === 'live')
            ? 'https://api.yourgateway.com/v1'
            : 'https://sandbox.yourgateway.com/v1';

        $ch = curl_init($apiEndpoint . '/charges/' . $params['transid'] . '/capture');

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $params['apiKey'],
            ],
            CURLOPT_RETURNTRANSFER => true,
        ]);

        $response = curl_exec($ch);
        $result = json_decode($response, true);
        curl_close($ch);

        if ($result['status'] === 'succeeded') {
            return [
                'status' => 'success',
                'transid' => $result['id'],
                'rawdata' => $result,
            ];
        }

        return [
            'status' => 'failed',
            'message' => $result['message'] ?? 'Capture failed',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'message' => $e->getMessage(),
        ];
    }
}
```

### Step 3: Create the Callback Handler

Create `callback/yourgateway.php`:

```php
<?php
/**
 * WHMCS Gateway Callback - YourGateway
 *
 * Handles payment notifications and callbacks
 */

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../../../includes/gatewayfunctions.php';
require_once __DIR__ . '/../../../includes/invoicefunctions.php';

define('GATEWAY_CALLBACK', true);

// Get gateway parameters
$gatewayParams = getGatewayVariables('yourgateway');

if (!$gatewayParams) {
    die("Gateway not found");
}

// Verify webhook signature
$payload = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_GATEWAY_SIGNATURE'] ?? '';
$expectedSignature = hash_hmac('sha256', $payload, $gatewayParams['webhookSecret']);

if (!hash_equals($expectedSignature, $signature)) {
    logTransaction($gatewayParams['paymentmethod'], $_REQUEST, 'Invalid Signature');
    http_response_code(403);
    die('Invalid signature');
}

// Parse webhook payload
$event = json_decode($payload, true);

if (json_last_error() !== JSON_ERROR_NONE) {
    logTransaction($gatewayParams['paymentmethod'], $payload, 'Invalid JSON');
    http_response_code(400);
    die('Invalid JSON');
}

// Process based on event type
switch ($event['type']) {
    case 'payment.success':
        $invoiceId = $event['data']['order_id'];
        $transactionId = $event['data']['transaction_id'];
        $amount = $event['data']['amount'];
        $fee = $event['data']['fee'] ?? 0;

        $transExists = checkCbTransID($transactionId);

        if (!$transExists) {
            $results = localApi('AddInvoicePayment', [
                'invoiceid' => $invoiceId,
                'transactionid' => $transactionId,
                'amount' => $amount,
                'fees' => $fee,
                'gateway' => 'yourgateway',
            ]);

            logTransaction($gatewayParams['paymentmethod'], $event, 'Success');

            if ($results['result'] === 'success') {
                echo 'OK';
            }
        }
        break;

    case 'payment.failed':
        logTransaction($gatewayParams['paymentmethod'], $event, 'Failed');
        break;

    case 'refund.created':
        $transactionId = $event['data']['transaction_id'];
        $refundAmount = $event['data']['refund_amount'];

        $results = localApi('AddRefund', [
            'transactionid' => $transactionId,
            'amount' => $refundAmount,
            'type' => 'refund',
            'gateway' => 'yourgateway',
            'notes' => 'Refunded via YourGateway',
        ]);

        logTransaction($gatewayParams['paymentmethod'], $event, 'Refunded');
        break;

    default:
        logTransaction($gatewayParams['paymentmethod'], $event, 'Unhandled Event: ' . $event['type']);
}

http_response_code(200);
echo 'OK';
```

### Step 4: Create Language File

Create `lang/english.php`:

```php
<?php

$_LANG = [
    'yourgateway' => 'Your Gateway Name',
    'yourgateway_description' => 'Pay securely with Your Gateway',
    'yourgateway_refund_success' => 'Refund processed successfully',
    'yourgateway_refund_failed' => 'Refund processing failed',
];
```

### Step 5: Install and Configure

1. Upload module to `/modules/gateways/yourgateway/`
2. Go to Configuration > System > Payment Gateways
3. Activate "Your Gateway Name"
4. Enter API credentials
5. Configure webhook URL in provider dashboard

## Expected Outcomes

- Gateway appears in WHMCS payment methods
- One-time payments process successfully
- Recurring payments work with card storage
- Webhooks update invoice status automatically
- Refunds process through WHMCS admin

## Testing Checklist

- [ ] Module installed and visible in gateways
- [ ] Configuration fields save correctly
- [ ] Payment form displays on checkout
- [ ] Test payments complete successfully
- [ ] Webhooks update invoice status
- [ ] Duplicate webhook calls handled
- [ ] Invalid signatures rejected
- [ ] Refund workflow functions
- [ ] Error logging captures failures
- [ ] PCI compliance requirements met
