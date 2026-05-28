# WHMCS Payment Gateway Setup Workflow

## Purpose

Set up and configure payment gateways in WHMCS, handle gateway integration, test payment processing, and ensure secure transaction handling.

## Prerequisites

- WHMCS admin access
- Payment gateway account credentials
- SSL certificate configured
- Webhook/Callback URL access
- API credentials from payment provider

## Workflow Steps

### Step 1: Research and Select Gateway

Evaluate and select appropriate payment gateway:

```php
// Common WHMCS-supported gateways
$gatewayOptions = [
    'stripe' => ['name' => 'Stripe', 'type' => 'card', 'recurring' => true],
    'paypal' => ['name' => 'PayPal', 'type' => 'card,paypal', 'recurring' => true],
    'braintree' => ['name' => 'Braintree', 'type' => 'card', 'recurring' => true],
    'authorizenet' => ['name' => 'Authorize.net', 'type' => 'card', 'recurring' => true],
    'adyen' => ['name' => 'Adyen', 'type' => 'card', 'recurring' => true],
    'square' => ['name' => 'Square', 'type' => 'card', 'recurring' => false]
];

// Consider factors:
// - Transaction fees
// - Recurring billing support
// - Multi-currency support
// - PCI compliance requirements
// - Settlement timeline
// - Integration complexity
```

### Step 2: Configure Gateway Module

Install and configure the payment gateway:

```php
// Method 1: Using WHMCS built-in gateway
// Navigate to: Configuration > System Settings > Payment Gateways
// Click "Activate" on the desired gateway
// Enter API credentials

// Method 2: Custom gateway module
// File: /modules/gateways/{gateway_name}.php

function {gateway}_config(): array {
    return [
        'FriendlyName' => ['value' => 'Gateway Display Name'],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'Found in your gateway dashboard'
        ],
        'api_secret' => [
            'FriendlyName' => 'API Secret',
            'Type' => 'password',
            'Size' => '50'
        ],
        'environment' => [
            'FriendlyName' => 'Environment',
            'Type' => 'dropdown',
            'Options' => 'sandbox,production',
            'Default' => 'sandbox',
            'Description' => 'Use sandbox for testing'
        ],
        'webhook_secret' => [
            'FriendlyName' => 'Webhook Secret',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'For verifying webhook signatures'
        ]
    ];
}
```

### Step 3: Implement Payment Link Function

Create the payment form:

```php
// File: /modules/gateways/{gateway_name}.php

function {gateway}_link(array $params): string {
    $invoiceId = $params['invoiceid'];
    $amount = number_format($params['amount'], 2, '.', '');
    $currency = $params['currency'] ?? 'USD';
    $clientEmail = $params['clientdetails']['email'];
    
    // Generate unique order reference
    $orderRef = 'INV-' . $invoiceId . '-' . time();
    
    // Build payment page URL or form
    // Option 1: Redirect to gateway hosted page
    $paymentUrl = ($params['environment'] === 'sandbox')
        ? 'https://sandbox.gateway.com/checkout'
        : 'https://api.gateway.com/checkout';
    
    $paramsArray = [
        'merchant_id' => $params['api_key'],
        'amount' => $amount,
        'currency' => $currency,
        'order_ref' => $orderRef,
        'customer_email' => $clientEmail,
        'success_url' => $params['systemurl'] . '/viewinvoice.php?id=' . $invoiceId . '&payment=success',
        'cancel_url' => $params['systemurl'] . '/viewinvoice.php?id=' . $invoiceId . '&payment=cancelled',
        'callback_url' => $params['systemurl'] . '/modules/gateways/callback/' . basename(__FILE__, '.php')
    ];
    
    // Create payment form
    $form = '<form action="' . $paymentUrl . '" method="POST" id="gateway_payment_form">';
    
    foreach ($paramsArray as $key => $value) {
        $form .= '<input type="hidden" name="' . htmlspecialchars($key) . '" value="' . htmlspecialchars($value) . '">';
    }
    
    $form .= '<button type="submit" class="btn btn-primary btn-lg">';
    $form .= '<i class="fa fa-credit-card"></i> Pay ' . $params['currency'] . ' ' . $amount;
    $form .= '</button>';
    $form .= '</form>';
    
    // Store order reference for callback verification
    Capsule::table('mod_gateway_pending')->insert([
        'invoice_id' => $invoiceId,
        'order_ref' => $orderRef,
        'amount' => $amount,
        'status' => 'pending',
        'created_at' => Capsule::raw('NOW()')
    ]);
    
    return $form;
}
```

### Step 4: Create Callback Handler

Handle payment callbacks and confirmations:

```php
// File: /modules/gateways/callback/{gateway_name}.php

// Verify webhook signature
function verifyWebhookSignature($payload, $signature, $secret) {
    $expected = hash_hmac('sha256', $payload, $secret);
    return hash_equals($expected, $signature);
}

// Process callback
add_hook('GatewayPaymentCallback', 1, function($vars) {
    $gatewayName = basename(__FILE__, '.php');
    
    // Get raw payload and signature
    $rawInput = file_get_contents('php://input');
    $signature = $_SERVER['HTTP_X_SIGNATURE'] ?? $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? '';
    
    // Verify signature
    $webhookSecret = Capsule::table('tblpaymentgateways')
        ->where('gateway', $gatewayName)
        ->where('setting', 'webhook_secret')
        ->value('value');
    
    if (!verifyWebhookSignature($rawInput, $signature, $webhookSecret)) {
        logTransaction($gatewayName, ['raw' => $rawInput], 'Invalid Signature');
        http_response_code(403);
        exit('Invalid signature');
    }
    
    // Parse webhook data
    $data = json_decode($rawInput, true);
    
    // Map gateway event types
    $eventType = $data['event_type'] ?? $data['type'] ?? '';
    
    switch ($eventType) {
        case 'payment.completed':
        case 'charge.succeeded':
            handleSuccessfulPayment($data, $gatewayName);
            break;
            
        case 'payment.failed':
        case 'charge.failed':
            handleFailedPayment($data, $gatewayName);
            break;
            
        case 'refund.created':
            handleRefund($data, $gatewayName);
            break;
            
        default:
            logTransaction($gatewayName, $data, 'Unknown Event: ' . $eventType);
    }
    
    echo 'OK';
});

function handleSuccessfulPayment($data, $gatewayName) {
    $transactionId = $data['transaction_id'] ?? $data['id'] ?? '';
    $amount = $data['amount'] ?? 0;
    $orderRef = $data['order_ref'] ?? $data['metadata']['order_ref'] ?? '';
    
    // Find pending transaction
    $pending = Capsule::table('mod_gateway_pending')
        ->where('order_ref', $orderRef)
        ->where('status', 'pending')
        ->first();
    
    if (!$pending) {
        logTransaction($gatewayName, $data, 'Pending Transaction Not Found');
        return;
    }
    
    // Verify amount matches
    if ((float)$amount !== (float)$pending->amount) {
        logTransaction($gatewayName, $data, 'Amount Mismatch');
        return;
    }
    
    // Add payment to WHMCS invoice
    addInvoicePayment(
        $pending->invoice_id,
        $transactionId,
        $amount,
        0,
        $gatewayName
    );
    
    // Update pending record
    Capsule::table('mod_gateway_pending')
        ->where('id', $pending->id)
        ->update([
            'status' => 'completed',
            'transaction_id' => $transactionId,
            'completed_at' => Capsule::raw('NOW()')
        ]);
    
    // Log transaction
    logTransaction($gatewayName, $data, 'Success');
}

function handleFailedPayment($data, $gatewayName) {
    $orderRef = $data['order_ref'] ?? '';
    $reason = $data['failure_reason'] ?? $data['error']['message'] ?? 'Unknown';
    
    $pending = Capsule::table('mod_gateway_pending')
        ->where('order_ref', $orderRef)
        ->first();
    
    if ($pending) {
        Capsule::table('mod_gateway_pending')
            ->where('id', $pending->id)
            ->update(['status' => 'failed', 'failure_reason' => $reason]);
        
        logTransaction($gatewayName, $data, 'Failed: ' . $reason);
    }
}
```

### Step 5: Configure Webhook URL

Set up webhook endpoint with payment provider:

```php
// Your callback URL should be:
// https://yourdomain.com/modules/gateways/callback/{gateway_name}.php

// Common webhook URL configurations:

// Stripe: Configure in Dashboard > Webhooks
// URL: https://yourdomain.com/modules/gateways/callback/stripe.php
// Events: payment_intent.succeeded, charge.refunded

// PayPal: Configure in Dashboard > Webhooks
// URL: https://yourdomain.com/modules/gateways/callback/paypal.php
// Events: PAYMENT.CAPTURE.COMPLETED, PAYMENT.CAPTURE.REFUNDED

// Braintree: Configure in Dashboard > Webhooks
// URL: https://yourdomain.com/modules/gateways/callback/braintree.php
// Events: settlement.confirmed, refund.succeeded

// Required webhook events to enable:
// - Payment completed/succeeded
// - Payment failed
// - Refund processed
// - Subscription updates (if applicable)
```

### Step 6: Implement Refund Capability

Add refund handling:

```php
// File: /modules/gateways/{gateway_name}/refund.php

function {gateway}_refund($params): array {
    $transactionId = $params['transid'];
    $amount = $params['amount'];
    
    $gatewayConfig = Capsule::table('tblpaymentgateways')
        ->where('gateway', 'stripe')
        ->pluck('value', 'setting')
        ->toArray();
    
    $stripe = new \Stripe\StripeClient($gatewayConfig['api_key']);
    
    try {
        // If transaction is a PaymentIntent
        $refund = $stripe->refunds->create([
            'charge' => $transactionId,
            'amount' => (int)($amount * 100)
        ]);
        
        if ($refund->status === 'succeeded') {
            logTransaction('stripe', [
                'original' => $transactionId,
                'refund' => $refund->id,
                'amount' => $amount
            ], 'Refunded');
            
            return ['status' => 'success', 'refund_id' => $refund->id];
        }
        
        return ['status' => 'failed', 'error' => 'Refund not completed'];
        
    } catch (\Stripe\Exception\CardException $e) {
        return ['status' => 'failed', 'error' => $e->getMessage()];
    }
}
```

### Step 7: Test Gateway Integration

Comprehensive testing checklist:

```php
// Test scenarios:

// 1. Sandbox Testing
// - Set gateway to sandbox mode
// - Use test card numbers provided by gateway
// - Test successful payment
// - Test failed payment
// - Test refund processing

// 2. Webhook Testing
// Use gateway's webhook testing tools:
// - Stripe: stripe listen --forward-to localhost:80/modules/gateways/callback/stripe.php
// - PayPal: Use webhook simulator
// - Braintree: Use sandbox webhook testing

// 3. Test Cases:

// Test successful payment
/*
POST /modules/gateways/callback/{gateway}.php
{
    "event_type": "payment.completed",
    "transaction_id": "ch_test123",
    "amount": 99.99,
    "order_ref": "INV-123-1234567890"
}
Expected: Invoice marked as paid, confirmation email sent
*/

// Test failed payment
/*
{
    "event_type": "payment.failed",
    "order_ref": "INV-123-1234567890",
    "failure_reason": "Card declined"
}
Expected: Transaction logged, admin notified if configured
*/

// Test refund
/*
{
    "event_type": "refund.created",
    "transaction_id": "re_test123",
    "original_transaction_id": "ch_test123",
    "amount": 50.00
}
Expected: Credit applied to client account
*/
```

### Step 8: Production Deployment

Move from sandbox to production:

```php
// Pre-deployment checklist:

// 1. Update gateway configuration
// Change environment from 'sandbox' to 'production'
// Update API keys from production credentials

// 2. Update webhook URL
// Ensure production webhook URL is configured
// Test webhook connectivity

// 3. SSL Certificate
// Verify SSL is properly configured
// Test HTTPS callbacks

// 4. Update WHMCS settings
// Set gateway as default if desired
// Enable for all currencies supported

// 5. Monitor initial transactions
// Watch first few transactions closely
// Verify settlement in gateway dashboard

// Database update for production:
UPDATE tblpaymentgateways 
SET value = 'production' 
WHERE gateway = '{gateway_name}' 
AND setting = 'environment';
```

## Verification Checklist

- [ ] Gateway module installed and activated
- [ ] API credentials configured correctly
- [ ] Test transactions completing successfully
- [ ] Webhooks receiving and processing
- [ ] Failed payments handled correctly
- [ ] Refund functionality working
- [ ] Email notifications sent correctly
- [ ] Transaction logs accurate
- [ ] Production credentials deployed
- [ ] Monitoring in place for issues

## Related Skills and Documentation

- [WHMCS Payment Processing](whmcs-payment-processing-workflow.md)
- [WHMCS Payment Reconciliation](whmcs-payment-reconciliation-workflow.md)
- [WHMCS Refund Workflow](whmcs-refund-workflow.md)
- WHMCS Documentation: Payment Gateway Development
- Payment Gateway Documentation: Webhook Integration

## Notes

- Always test in sandbox before production
- Keep API credentials secure and rotated regularly
- Implement proper webhook signature verification
- Monitor for failed transactions and investigate
- Document gateway-specific requirements
- Plan for gateway downtime scenarios
- Review PCI compliance requirements
- Maintain transaction logs for audit