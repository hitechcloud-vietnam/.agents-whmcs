# WHMCS Stripe Connect Setup Workflow

## Description
Configure Stripe Connect for WHMCS marketplace/vendor payments.

## Prerequisites
- Stripe account
- Stripe Connect application
- WHMCS 7.0+

## Steps

### Step 1: Create Stripe Connect Application
```bash
# In Stripe Dashboard:
# 1. Go to Settings > Connect > Manage > New Application
# 2. Set application type
# 3. Get client_id and secret
```

### Step 2: Configure WHMCS Gateway
```php
<?php
// In configuration.php

$stripe_config = [
    'client_id' => 'ca_xxxxx',
    'client_secret' => 'sk_live_xxxxx',
    'webhook_secret' => 'whsec_xxxxx',
    'mode' => 'live', // or 'test'
];
```

### Step 3: Create Stripe Connect Gateway
```php
<?php
// modules/gateways/stripe_connect/stripe_connect.php

function stripe_connect_MetaData()
{
    return [
        'DisplayName' => 'Stripe Connect',
        'APIVersion' => '1.0',
    ];
}

function stripe_connect_config()
{
    return [
        'FriendlyName' => ['value' => 'Stripe Connect'],
        'publishableKey' => ['Type' => 'text', 'Label' => 'Publishable Key'],
        'secretKey' => ['Type' => 'password', 'Label' => 'Secret Key'],
        'webhookSecret' => ['Type' => 'password', 'Label' => 'Webhook Secret'],
        'applicationFee' => ['Type' => 'text', 'Label' => 'Application Fee (%)'],
    ];
}

function stripe_connect_link($params)
{
    $stripe = new \Stripe\Stripe($params['secretKey']);
    
    $session = \Stripe\Checkout\Session::create([
        'payment_method_types' => ['card'],
        'line_items' => [[
            'price_data' => [
                'currency' => strtolower($params['currency']),
                'product_data' => [
                    'name' => 'Invoice #' . $params['invoiceid'],
                ],
                'unit_amount' => (int)($params['amount'] * 100),
            ],
            'quantity' => 1,
        ]],
        'mode' => 'payment',
        'success_url' => $params['returnurl'],
        'cancel_url' => $params['returnurl'],
        'metadata' => [
            'invoice_id' => $params['invoiceid'],
            'client_id' => $params['clientdetails']['id'],
        ],
    ]);
    
    return '<script src="https://js.stripe.com/v3/"></script>
        <button id="stripe-button">Pay with Stripe</button>
        <script>
            var stripe = Stripe("' . $params['publishableKey'] . '");
            document.getElementById("stripe-button").onclick = function() {
                stripe.redirectToCheckout({ sessionId: "' . $session->id . '" });
            };
        </script>';
}
```

### Step 3: Handle Webhooks
```php
<?php
// modules/gateways/stripe_connect/callback.php

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../../../includes/gatewayfunctions.php';

$gateway = getGatewayVariables('stripe_connect');

// Verify webhook signature
$payload = file_get_contents('php://input');
$sig = $_SERVER['HTTP_STRIPE_SIGNATURE'];
$event = \Stripe\Webhook::constructEvent(
    $payload, $sig, $gateway['webhookSecret']
);

switch ($event->type) {
    case 'checkout.session.completed':
        $session = $event->data->object;
        $invoiceId = $session->metadata->invoice_id;
        $amount = $session->amount_total / 100;
        
        addInvoicePayment($invoiceId, $session->payment_intent, $amount, 0, 'stripe_connect');
        break;
        
    case 'payment_intent.payment_failed':
        // Handle failure
        break;
}
```

### Step 4: Connect Vendor Accounts (Marketplace)
```php
<?php
// Onboard vendor
function createConnectAccount($vendorEmail, $vendorName)
{
    $account = \Stripe\Account::create([
        'type' => 'express',
        'email' => $vendorEmail,
        'capabilities' => [
            'card_payments' => ['requested' => true],
            'transfers' => ['requested' => true],
        ],
        'business_type' => 'individual',
        'individual' => ['first_name' => $vendorName],
    ]);
    
    // Create account link for onboarding
    $link = \Stripe\AccountLink::create([
        'account' => $account->id,
        'refresh_url' => 'https://yoursite.com/onboard-refresh',
        'return_url' => 'https://yoursite.com/onboard-complete',
        'type' => 'account_onboarding',
    ]);
    
    return $link->url;
}

// Create transfer to vendor
function transferToVendor($amount, $vendorAccountId, $invoiceId)
{
    $fee = $amount * 0.05; // 5% platform fee
    
    \Stripe\Transfer::create([
        'amount' => (int)(($amount - $fee) * 100),
        'currency' => 'usd',
        'destination' => $vendorAccountId,
        'metadata' => ['invoice_id' => $invoiceId],
    ]);
}
```

### Step 5: Test Integration
```bash
# Use Stripe test mode
# Create test vendor accounts
# Process test payments
# Verify webhook delivery
# Check transfer calculations
```

## Tags
- stripe
- payment
- marketplace
- connect