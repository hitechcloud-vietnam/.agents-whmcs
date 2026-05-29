# WHMCS Payment Gateway Setup Workflow

## Overview
This workflow covers implementing and configuring payment gateways in WHMCS.

## Step 1: Payment Gateway Module

```php
<?php
// modules/gateways/your_gateway.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Gateway configuration
 */
function your_gateway_MetaData()
{
    return [
        'DisplayName' => 'Your Payment Gateway',
        'APIVersion' => '1.0',
        'DisableLocalCreditCardInput' => false,
        'TokenisedStorage' => false,
    ];
}

function your_gateway_config()
{
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'Your Payment Gateway'],
        'apiKey' => ['Type' => 'password', 'Description' => 'Your API Key'],
        'environment' => [
            'Type' => 'dropdown',
            'Options' => 'Production,Sandbox',
            'Description' => 'Select environment'
        ],
        'webhookSecret' => ['Type' => 'password', 'Description' => 'Webhook secret for verification']
    ];
}

/**
 * Link to payment page
 */
function your_gateway_link($params)
{
    $apiKey = $params['apiKey'];
    $environment = $params['environment'];
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    $clientEmail = $params['clientdetails']['email'];

    // Create payment session
    $sessionId = createPaymentSession($apiKey, $environment, $invoiceId, $amount, $currency);

    // Build payment URL
    $paymentUrl = ($environment === 'Sandbox')
        ? 'https://sandbox.yourgateway.com/pay'
        : 'https://api.yourgateway.com/pay';

    $html = '<form method="POST" action="' . $paymentUrl . '">';
    $html .= '<input type="hidden" name="session_id" value="' . $sessionId . '">';
    $html .= '<input type="hidden" name="return_url" value="' . $params['returnurl'] . '">';
    $html .= '<input type="hidden" name="cancel_url" value="' . $params['cancelurl'] . '">';
    $html .= '<button type="submit" class="btn btn-primary">Pay with Your Gateway</button>';
    $html .= '</form>';

    return $html;
}

/**
 * Refund transaction
 */
function your_gateway_refund($params)
{
    $apiKey = $params['apiKey'];
    $transactionId = $params['transid'];
    $amount = $params['amount'];
    $environment = $params['environment'];

    try {
        $response = callGatewayApi($apiKey, $environment, 'POST', '/refunds', [
            'transaction_id' => $transactionId,
            'amount' => $amount
        ]);

        return [
            'status' => 'success',
            'transid' => $response['refund_id'],
            'rawdata' => $response
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'errormsg' => $e->getMessage()
        ];
    }
}

/**
 * Storn function
 */
function your_gateway_storn($params)
{
    return your_gateway_refund($params);
}

/**
 * Verify callback
 */
function your_gateway_verify_callback($params)
{
    $webhookSecret = $params['webhookSecret'];

    // Verify webhook signature
    $signature = $_SERVER['HTTP_X_GATEWAY_SIGNATURE'] ?? '';
    $payload = file_get_contents('php://input');

    $expected = hash_hmac('sha256', $payload, $webhookSecret);
    if (!hash_equals($expected, $signature)) {
        return ['error' => 'Invalid signature'];
    }

    $data = json_decode($payload, true);

    // Process payment
    if ($data['event'] === 'payment.completed') {
        addInvoicePayment(
            $data['invoice_id'],
            $data['amount'],
            $data['transaction_id'],
            'your_gateway'
        );
    }

    return ['success' => true];
}

function createPaymentSession(string $apiKey, string $environment, int $invoiceId, float $amount, string $currency): string
{
    $baseUrl = ($environment === 'Sandbox')
        ? 'https://sandbox.yourgateway.com'
        : 'https://api.yourgateway.com';

    $ch = curl_init("$baseUrl/sessions");
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_HTTPHEADER => [
            'Authorization: Bearer ' . $apiKey,
            'Content-Type: application/json'
        ],
        CURLOPT_POSTFIELDS => json_encode([
            'invoice_id' => $invoiceId,
            'amount' => $amount,
            'currency' => $currency
        ]),
        CURLOPT_RETURNTRANSFER => true
    ]);

    $response = curl_exec($ch);
    curl_close($ch);

    $result = json_decode($response, true);
    return $result['session_id'];
}

function callGatewayApi(string $apiKey, string $environment, string $method, string $endpoint, array $data = []): array
{
    $baseUrl = ($environment === 'Sandbox')
        ? 'https://sandbox.yourgateway.com'
        : 'https://api.yourgateway.com';

    $ch = curl_init($baseUrl . $endpoint);
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            'Authorization: Bearer ' . $apiKey,
            'Content-Type: application/json'
        ]
    ]);

    if ($method === 'POST') {
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
    }

    $response = curl_exec($ch);
    curl_close($ch);

    return json_decode($response, true);
}
```

## Verification Checklist

- [ ] Gateway module created
- [ ] Configuration fields set up
- [ ] Payment link generation working
- [ ] Refund function implemented
- [ ] Webhook callback working
- [ ] Test payment successful
