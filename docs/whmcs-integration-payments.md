# WHMCS Payment Gateway Integration

Complete guide for integrating WHMCS with payment processors.

## Overview

Connect WHMCS to various payment providers for accepting online payments.

## Stripe Integration

### Stripe Payment Gateway

```php
<?php
/**
 * Stripe payment gateway module
 */
function stripe_config(): array
{
    return [
        'FriendlyName' => ['value' => 'Stripe'],
        'publishableKey' => [
            'FriendlyName' => 'Publishable Key',
            'Type' => 'text',
        ],
        'secretKey' => [
            'FriendlyName' => 'Secret Key',
            'Type' => 'password',
        ],
        'webhookSecret' => [
            'FriendlyName' => 'Webhook Secret',
            'Type' => 'password',
        ],
    ];
}

/**
 * Stripe payment link
 */
function stripe_link(array $params): string
{
    $apiKey = $params['config']['secretKey'];
    
    // Create checkout session
    $ch = curl_init('https://api.stripe.com/v1/checkout/sessions');
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => http_build_query([
            'payment_method_types[]' => 'card',
            'line_items[][price_data][currency]' => strtolower($params['currency']),
            'line_items[][price_data][unit_amount]' => (int)($params['amount'] * 100),
            'line_items[][price_data][product_data][name]' => 'Invoice #' . $params['invoiceid'],
            'mode' => 'payment',
            'success_url' => $params['returnurl'] . '&session_id={CHECKOUT_SESSION_ID}',
            'cancel_url' => $params['returnurl'],
            'client_reference_id' => $params['invoiceid'],
            'metadata' => [
                'invoice_id' => $params['invoiceid'],
                'client_id' => $params['clientdetails']['id'],
            ],
        ]),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            'Authorization: Bearer ' . $apiKey,
            'Content-Type: application/x-www-form-urlencoded',
        ],
    ]);
    
    $response = json_decode(curl_exec($ch), true);
    curl_close($ch);
    
    if (isset($response['id'])) {
        return '<form action="' . $response['url'] . '" method="GET">
            <button type="submit">Pay with Stripe</button>
        </form>';
    }
    
    return 'Error creating checkout session';
}
```

### Stripe Webhook Handler

```php
<?php
/**
 * modules/gateways/callback/stripe.php
 */

// Set up response
header('Content-Type: application/json');

// Get webhook payload
$payload = @file_get_contents('php://input');
$event = json_decode($payload, true);

// Verify signature
$signature = $_SERVER['HTTP_STRIPE_SIGNATURE'];
$webhookSecret = $gatewayParams['config']['webhookSecret'];

try {
    $sigHeader = $signature;
    $event = \Stripe\Webhook::constructEvent(
        $payload,
        $sigHeader,
        $webhookSecret
    );
} catch (Exception $e) {
    http_response_code(400);
    exit('Webhook Error: ' . $e->getMessage());
}

// Handle events
switch ($event['type']) {
    case 'checkout.session.completed':
        $session = $event['data']['object'];
        
        $invoiceId = $session['client_reference_id'];
        $transactionId = $session['payment_intent'];
        $amount = $session['amount_total'] / 100;
        
        // Log transaction
        logTransaction('stripe', [
            'invoice_id' => $invoiceId,
            'transaction_id' => $transactionId,
            'amount' => $amount,
            'currency' => $session['currency'],
        ], 'Payment');
        
        // Add payment to invoice
        addInvoicePayment($invoiceId, $transactionId, $amount, 0, 'stripe');
        
        // Redirect to success
        header('Location: ' . $systemurl . '/viewinvoice.php?id=' . $invoiceId . '&paymentsuccess=true');
        exit;
        
    case 'payment_intent.payment_failed':
        $intent = $event['data']['object'];
        logTransaction('stripe', $intent, 'Failed');
        break;
}

http_response_code(200);
echo json_encode(['status' => 'received']);
```

## PayPal Integration

```php
<?php
/**
 * PayPal Express Checkout
 */
function paypal_config(): array
{
    return [
        'FriendlyName' => ['value' => 'PayPal Express'],
        'clientId' => ['FriendlyName' => 'Client ID', 'Type' => 'text'],
        'secret' => ['FriendlyName' => 'Secret', 'Type' => 'password'],
        'environment' => ['FriendlyName' => 'Environment', 'Type' => 'dropdown', 
            'Options' => 'Live,Sandbox'],
    ];
}

/**
 * Get PayPal access token
 */
function getPayPalAccessToken(string $clientId, string $secret, bool $sandbox): string
{
    $baseUrl = $sandbox 
        ? 'https://api-m.sandbox.paypal.com' 
        : 'https://api-m.paypal.com';
    
    $ch = curl_init($baseUrl . '/v1/oauth2/token');
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => 'grant_type=client_credentials',
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_USERPWD => "{$clientId}:{$secret}",
        CURLOPT_HTTPHEADER => ['Content-Type: application/x-www-form-urlencoded'],
    ]);
    
    $response = json_decode(curl_exec($ch), true);
    curl_close($ch);
    
    return $response['access_token'];
}

/**
 * Create PayPal order
 */
function paypal_link(array $params): string
{
    $accessToken = getPayPalAccessToken(
        $params['config']['clientId'],
        $params['config']['secret'],
        $params['config']['environment'] === 'Sandbox'
    );
    
    $baseUrl = $params['config']['environment'] === 'Sandbox'
        ? 'https://api-m.sandbox.paypal.com'
        : 'https://api-m.paypal.com';
    
    $orderData = [
        'intent' => 'CAPTURE',
        'purchase_units' => [[
            'reference_id' => $params['invoiceid'],
            'amount' => [
                'currency_code' => $params['currency'],
                'value' => number_format($params['amount'], 2, '.', ''),
            ],
            'description' => 'Invoice #' . $params['invoiceid'],
        ]],
        'application_context' => [
            'return_url' => $params['returnurl'],
            'cancel_url' => $params['returnurl'],
        ],
    ];
    
    $ch = curl_init($baseUrl . '/v2/checkout/orders');
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode($orderData),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            'Authorization: Bearer ' . $accessToken,
            'Content-Type: application/json',
        ],
    ]);
    
    $response = json_decode(curl_exec($ch), true);
    curl_close($ch);
    
    // Find approve URL
    $approveUrl = '';
    foreach ($response['links'] as $link) {
        if ($link['rel'] === 'approve') {
            $approveUrl = $link['href'];
            break;
        }
    }
    
    return '<form action="' . $approveUrl . '" method="POST">
        <button type="submit">Pay with PayPal</button>
    </form>';
}

/**
 * Capture PayPal payment
 */
function capturePayPalPayment(string $orderId, array $params): array
{
    $accessToken = getPayPalAccessToken(
        $params['config']['clientId'],
        $params['config']['secret'],
        $params['config']['environment'] === 'Sandbox'
    );
    
    $baseUrl = $params['config']['environment'] === 'Sandbox'
        ? 'https://api-m.sandbox.paypal.com'
        : 'https://api-m.paypal.com';
    
    $ch = curl_init($baseUrl . "/v2/checkout/orders/{$orderId}/capture");
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            'Authorization: Bearer ' . $accessToken,
            'Content-Type: application/json',
        ],
    ]);
    
    $response = json_decode(curl_exec($ch), true);
    curl_close($ch);
    
    if ($response['status'] === 'COMPLETED') {
        return [
            'success' => true,
            'transaction_id' => $response['purchase_units'][0]['payments']['captures'][0]['id'],
            'amount' => $response['purchase_units'][0]['payments']['captures'][0]['amount']['value'],
        ];
    }
    
    return ['success' => false, 'error' => 'Payment not completed'];
}
```

## Square Integration

```php
<?php
/**
 * Square payment gateway
 */
function square_config(): array
{
    return [
        'FriendlyName' => ['value' => 'Square'],
        'applicationId' => ['FriendlyName' => 'Application ID', 'Type' => 'text'],
        'accessToken' => ['FriendlyName' => 'Access Token', 'Type' => 'password'],
        'locationId' => ['FriendlyName' => 'Location ID', 'Type' => 'text'],
        'environment' => ['FriendlyName' => 'Environment', 'Type' => 'dropdown',
            'Options' => 'Production,Sandbox'],
    ];
}

/**
 * Create Square payment
 */
function square_link(array $params): string
{
    $env = $params['config']['environment'] === 'Sandbox' ? 'sandbox' : 'production';
    $baseUrl = "https://connect.{$env}.squareup.com";
    
    // Create payment
    $paymentData = [
        'source_id' => $_POST['sourceId'],
        'idempotency_key' => uniqid(),
        'amount_money' => [
            'amount' => (int)($params['amount'] * 100),
            'currency' => $params['currency'],
        ],
        'location_id' => $params['config']['locationId'],
        'reference_id' => $params['invoiceid'],
    ];
    
    $ch = curl_init($baseUrl . '/v2/payments');
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode($paymentData),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            'Authorization: Bearer ' . $params['config']['accessToken'],
            'Content-Type: application/json',
            'Square-Version: 2024-01-18',
        ],
    ]);
    
    $response = json_decode(curl_exec($ch), true);
    curl_close($ch);
    
    if (isset($response['payment'])) {
        logTransaction('square', $response['payment'], 'Payment');
        addInvoicePayment(
            $params['invoiceid'],
            $response['payment']['id'],
            $params['amount'],
            0,
            'square'
        );
    }
    
    return 'Payment processed';
}
```

## Payment Webhooks

```php
<?php
/**
 * Handle payment gateway callbacks
 */
function handlePaymentCallback(string $gateway): array
{
    $params = $_POST;
    
    // Log raw data
    logTransaction($gateway, $params, 'Callback received');
    
    // Validate and process based on gateway
    switch ($gateway) {
        case 'stripe':
            return handleStripeCallback($params);
        case 'paypal':
            return handlePayPalCallback($params);
        case 'square':
            return handleSquareCallback($params);
        default:
            return ['error' => 'Unknown gateway'];
    }
}
```

## Best Practices

1. **Use webhooks** - For reliable payment notifications
2. **Verify signatures** - Always validate payment signatures
3. **Handle idempotency** - Prevent duplicate payments
4. **Log all transactions** - Keep detailed audit trail
5. **Support refunds** - Implement refund functionality
6. **Test in sandbox** - Verify before production

## Related Documentation

- [whmcs-module-gateway-api.md](whmcs-module-gateway-api.md)
- [whmcs-integration-api.md](whmcs-integration-api.md)
