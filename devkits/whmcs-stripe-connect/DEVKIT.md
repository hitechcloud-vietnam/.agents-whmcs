# WHMCS Stripe Connect Gateway Module - DEVKIT

## Module Information
- **Name**: Stripe Connect
- **Version**: 1.0.0
- **Type**: Gateway Module
- **Description**: Stripe Connect payment gateway with split payments support

## Installation
1. Copy to `/modules/gateways/stripe_connect/`
2. Activate via WHMCS Admin > Configuration > Payment Gateways

## stripe_connect.php
```php
<?php
/**
 * WHMCS Stripe Connect Payment Gateway
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function stripe_connect_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Stripe Connect'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'Stripe Connect with split payments and subscriptions'
        ],
        'stripe_secret_key' => [
            'FriendlyName' => 'Secret Key',
            'Type' => 'password',
            'Size' => '80',
            'Description' => 'Stripe secret key (sk_live_...)'
        ],
        'stripe_publishable_key' => [
            'FriendlyName' => 'Publishable Key',
            'Type' => 'text',
            'Size' => '80',
            'Description' => 'Stripe publishable key (pk_live_...)'
        ],
        'stripe_webhook_secret' => [
            'FriendlyName' => 'Webhook Secret',
            'Type' => 'password',
            'Size' => '80',
            'Description' => 'Stripe webhook signing secret (whsec_...)'
        ],
        'stripe_currency' => [
            'FriendlyName' => 'Currency',
            'Type' => 'text',
            'Size' => '10',
            'Default' => 'USD',
            'Description' => 'Default currency code'
        ],
        'application_fee_percent' => [
            'FriendlyName' => 'Application Fee (%)',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '0',
            'Description' => 'Percentage fee for platform'
        ],
        'stripe_statement_descriptor' => [
            'FriendlyName' => 'Statement Descriptor',
            'Type' => 'text',
            'Size' => '22',
            'Description' => 'Appears on customer statements'
        ]
    ];
}

function stripe_connect_capture($params)
{
    $stripeSecret = $params['stripe_secret_key'];
    $amount = (int)round($params['amount'] * 100); // Convert to cents
    $currency = strtolower($params['currency']);
    $invoiceId = $params['invoiceid'];
    $clientEmail = $params['clientdetails']['email'] ?? '';
    $description = 'Payment for Invoice #' . $invoiceId;
    
    // Get payment method from POST
    $paymentMethodId = $_POST['stripe_payment_method'] ?? '';
    
    if (empty($paymentMethodId)) {
        return [
            'status' => 'failed',
            'error' => 'No payment method provided'
        ];
    }
    
    // Create PaymentIntent
    $payload = [
        'amount' => $amount,
        'currency' => $currency,
        'payment_method' => $paymentMethodId,
        'confirmation_method' => 'manual',
        'confirm' => true,
        'description' => $description,
        'receipt_email' => $clientEmail,
        'metadata' => [
            'invoice_id' => $invoiceId,
            'order_id' => $params['orderid'] ?? 0
        ]
    ];
    
    // Add application fee if configured
    $applicationFeePercent = $params['application_fee_percent'] ?? 0;
    if ($applicationFeePercent > 0) {
        $applicationFee = (int)round($amount * $applicationFeePercent / 100);
        $payload['application_fee_amount'] = $applicationFee;
    }
    
    $ch = curl_init('https://api.stripe.com/v1/payment_intents');
    curl_setopt($ch, CURLOPT_USERPWD, $stripeSecret . ':');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['error'])) {
        return [
            'status' => 'failed',
            'error' => $result['error']['message'] ?? 'Payment failed'
        ];
    }
    
    if (isset($result['status']) && $result['status'] == 'requires_action') {
        return [
            'status' => 'pending',
            'declined' => false,
            'pending' => true,
            'rawsuccess' => true,
            'clientsecret' => $result['client_secret'],
            'reference' => $result['id']
        ];
    }
    
    if (isset($result['status']) && $result['status'] == 'succeeded') {
        return [
            'status' => 'success',
            'transid' => $result['id'],
            'amount' => $result['amount'] / 100,
            'rawdata' => $response
        ];
    }
    
    return [
        'status' => 'pending',
        'pending' => true,
        'clientsecret' => $result['client_secret'],
        'reference' => $result['id']
    ];
}

function stripe_connect_callback($params)
{
    $stripeSecret = $params['stripe_secret_key'];
    $webhookSecret = $params['stripe_webhook_secret'];
    $payload = file_get_contents('php://input');
    $sigHeader = $_SERVER['HTTP_STRIPE_SIGNATURE'] ?? '';
    
    // Verify webhook signature
    $elements = explode(',', $sigHeader);
    $timestamp = '';
    $signatures = [];
    
    foreach ($elements as $element) {
        if (strpos($element, 't=') === 0) {
            $timestamp = str_replace('t=', '', $element);
        }
        if (strpos($element, 'v1=') === 0) {
            $signatures[] = str_replace('v1=', '', $element);
        }
    }
    
    // Compute expected signature
    $computedSig = hash_hmac('sha256', $timestamp . '.' . $payload, $webhookSecret);
    
    if (!in_array($computedSig, $signatures)) {
        return ['status' => 'error', 'rawdata' => 'Invalid signature'];
    }
    
    $event = json_decode($payload, true);
    
    if ($event['type'] == 'payment_intent.succeeded') {
        $paymentIntent = $event['data']['object'];
        
        return [
            'status' => 'success',
            'transid' => $paymentIntent['id'],
            'amount' => $paymentIntent['amount'] / 100,
            'rawdata' => $payload
        ];
    }
    
    if ($event['type'] == 'payment_intent.payment_failed') {
        $paymentIntent = $event['data']['object'];
        
        return [
            'status' => 'declined',
            'error' => $paymentIntent['last_payment_error']['message'] ?? 'Payment failed',
            'rawdata' => $payload
        ];
    }
    
    return ['status' => 'pending'];
}

function stripe_connect_refund($params)
{
    $stripeSecret = $params['stripe_secret_key'];
    $transactionId = $params['transactionId'];
    $amount = (int)round($params['amount'] * 100);
    
    $ch = curl_init('https://api.stripe.com/v1/refunds');
    curl_setopt($ch, CURLOPT_USERPWD, $stripeSecret . ':');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query([
        'payment_intent' => $transactionId,
        'amount' => $amount
    ]));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['id'])) {
        return [
            'status' => 'success',
            'refund_id' => $result['id']
        ];
    } else {
        return [
            'status' => 'failed',
            'error' => $result['error']['message'] ?? 'Refund failed'
        ];
    }
}

function stripe_connect_query($params)
{
    $stripeSecret = $params['stripe_secret_key'];
    $transactionId = $params['reference'];
    
    $ch = curl_init('https://api.stripe.com/v1/payment_intents/' . $transactionId);
    curl_setopt($ch, CURLOPT_USERPWD, $stripeSecret . ':');
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

add_hook('InvoicePaid', 1, function($vars) {
    if ($vars['paymentmethod'] == 'stripe_connect') {
        logActivity("Stripe Connect payment confirmed: " . ($vars['transid'] ?? 'N/A'));
    }
});

add_hook('AdminHomepage', 1, function($vars) {
    return [
        'stripeRevenue' => StripeConnectHelper::getTodayRevenue()
    ];
});

class StripeConnectHelper
{
    public static function getTodayRevenue()
    {
        $result = full_query("
            SELECT COALESCE(SUM(amountin), 0) as revenue
            FROM " . TABLE_PREFIX . "tblaccounts
            WHERE paymentmethod = 'stripe_connect'
            AND date = CURDATE()
        ");
        
        $row = mysql_fetch_array($result);
        return $row['revenue'] ?? 0;
    }
}
```