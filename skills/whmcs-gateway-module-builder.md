# WHMCS Gateway Module Builder

## Concept

A WHMCS gateway module enables payment processing through various payment providers. Gateway modules handle the entire payment lifecycle including payment form display, callback processing, and refund handling.

## File Structure

```
/modules/gateways/
├── yourgateway/
│   ├── yourgateway.php          # Main gateway file
│   ├── callback.php             # Payment callback handler
│   ├── refund.php               # Refund processing (optional)
│   └── templates/
│       ├── form.tpl             # Payment form template
│       └── receipt.tpl          # Receipt template
```

## Core Gateway Structure

```php
<?php
/**
 * Gateway Module: Your Gateway
 * Version: 1.0.0
 * Description: Payment gateway for...
 * Author: Your Name
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Gateway configuration options
 */
function yourgateway_MetaData()
{
    return [
        'DisplayName' => 'Your Gateway',
        'APIVersion' => '1.0',
        'OfflineCreditCard' => false,
        'AdminServiceFeePercentage' => false,
    ];
}

/**
 * Configuration array
 */
function yourgateway_config(array $params)
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Gateway',
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'Your gateway API key',
        ],
        'merchant_id' => [
            'FriendlyName' => 'Merchant ID',
            'Type' => 'text',
            'Size' => '30',
        ],
        'environment' => [
            'FriendlyName' => 'Environment',
            'Type' => 'dropdown',
            'Options' => [
                'sandbox' => 'Sandbox',
                'production' => 'Production',
            ],
            'Default' => 'sandbox',
        ],
        'webhook_secret' => [
            'FriendlyName' => 'Webhook Secret',
            'Type' => 'password',
            'Description' => 'Secret for webhook signature verification',
        ],
    ];
}

/**
 * Link to payment page
 */
function yourgateway_link(array $params)
{
    // Store params for callback
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    
    // Generate unique transaction reference
    $reference = 'INV' . $invoiceId . '_' . time();
    
    // Store in database for verification
    insert_query('tblgatewaytransactions', [
        'invoice_id' => $invoiceId,
        'gateway' => 'yourgateway',
        'reference' => $reference,
        'amount' => $amount,
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Smarty variables for template
    $gatewayUrl = ($params['environment'] === 'sandbox')
        ? 'https://sandbox.yourgateway.com/pay'
        : 'https://api.yourgateway.com/pay';
    
    return [
        'rawPostData' => '',
        'type' => 'Invoices',
        'params' => [
            'api_key' => $params['api_key'],
            'merchant_id' => $params['merchant_id'],
            'amount' => $amount,
            'currency' => $currency,
            'order_id' => $invoiceId,
            'reference' => $reference,
            'return_url' => $params['returnurl'],
            'cancel_url' => $params['returnurl'] . '&cancel=true',
            'notify_url' => $params['systemurl'] . 'modules/gateways/yourgateway/callback.php',
            'customer' => [
                'email' => $params['clientdetails']['email'],
                'name' => $params['clientdetails']['firstname'] . ' ' . $params['clientdetails']['lastname'],
            ],
        ],
        'direct' => true,
        'card_update' => false,
    ];
}
```

## Callback Handler

```php
<?php
/**
 * Callback handler for Your Gateway
 */

if (!defined("WHMCS")) {
    require_once __DIR__ . '/../../../../init.php';
}

$gateway = App::getFromRequest('gateway');
$invoiceId = App::getFromRequest('invoice_id');
$status = App::getFromRequest('status');
$transactionId = App::getFromRequest('transaction_id');
$amount = App::getFromRequest('amount');
$signature = App::getFromRequest('signature');

// Verify webhook signature
$expectedSignature = hash_hmac('sha256', $invoiceId . $transactionId . $amount, $gatewayParams['webhook_secret']);

if ($signature !== $expectedSignature) {
    logTransaction('yourgateway', $_REQUEST, 'Invalid Signature');
    http_response_code(403);
    exit('Invalid signature');
}

// Process based on status
switch ($status) {
    case 'success':
        $success = true;
        $failedReason = '';
        $transactionStatus = 'Paid';
        break;
    case 'failed':
        $success = false;
        $failedReason = App::getFromRequest('error_message');
        $transactionStatus = 'Failed';
        break;
    case 'pending':
        $success = true;
        $failedReason = '';
        $transactionStatus = 'Pending';
        break;
    default:
        logTransaction('yourgateway', $_REQUEST, 'Unknown Status');
        exit('Unknown status');
}

// Update transaction record
update_query('tblgatewaytransactions', [
    'status' => strtolower($status),
    'transaction_id' => $transactionId,
    'updated_at' => date('Y-m-d H:i:s'),
], [
    'reference' => App::getFromRequest('reference'),
]);

// Add invoice payment if successful
if ($status === 'success') {
    addInvoicePayment(
        $invoiceId,
        $transactionId,
        $amount,
        0,
        'yourgateway'
    );
}

logTransaction('yourgateway', $_REQUEST, $transactionStatus);
echo 'OK';
```

## Payment Form Template (form.tpl)

```smarty
<div class="yourgateway-form">
    <form action="{$gatewayUrl}" method="POST" id="payment-form">
        <input type="hidden" name="api_key" value="{$api_key}">
        <input type="hidden" name="merchant_id" value="{$merchant_id}">
        <input type="hidden" name="amount" value="{$amount}">
        <input type="hidden" name="currency" value="{$currency}">
        <input type="hidden" name="order_id" value="{$invoiceid}">
        <input type="hidden" name="reference" value="{$reference}">
        <input type="hidden" name="return_url" value="{$return_url}">
        
        <div class="form-group">
            <label for="card_number">Card Number</label>
            <input type="text" id="card_number" name="card_number" 
                   placeholder="1234 5678 9012 3456" 
                   class="form-control" required>
        </div>
        
        <div class="form-row">
            <div class="form-group">
                <label for="expiry">Expiry Date</label>
                <input type="text" id="expiry" name="expiry" 
                       placeholder="MM/YY" class="form-control" required>
            </div>
            <div class="form-group">
                <label for="cvv">CVV</label>
                <input type="text" id="cvv" name="cvv" 
                       placeholder="123" class="form-control" required>
            </div>
        </div>
        
        <div class="form-group">
            <label for="card_name">Name on Card</label>
            <input type="text" id="card_name" name="card_name" 
                   class="form-control" required>
        </div>
        
        <button type="submit" class="btn btn-primary btn-block">
            Pay {$amount} {$currency}
        </button>
    </form>
</div>

<script>
$(function() {
    $('#payment-form').on('submit', function() {
        // Collect card data
        var cardData = {
            number: $('#card_number').val(),
            expiry: $('#expiry').val(),
            cvv: $('#cvv').val(),
        };
        
        // Tokenize with gateway before submit
        YourGateway.tokenize(cardData, function(token) {
            $('<input>').attr({
                type: 'hidden',
                name: 'card_token',
                value: token
            }).appendTo('#payment-form');
            $('#payment-form')[0].submit();
        });
        
        return false;
    });
});
</script>
```

## Capture Function (for Authorize/Capture flows)

```php
function yourgateway_capture(array $params)
{
    try {
        $response = YourGatewayAPI::capture([
            'transaction_id' => $params['transid'],
            'amount' => $params['amount'],
            'currency' => $params['currency'],
        ]);
        
        if ($response['success']) {
            return [
                'success' => true,
                'transid' => $response['capture_id'],
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['error'],
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}
```

## Refund Function

```php
function yourgateway_refund(array $params)
{
    try {
        $response = YourGatewayAPI::refund([
            'original_transaction_id' => $params['transid'],
            'amount' => $params['amount'],
            'reason' => $params['reason'] ?? '',
        ]);
        
        if ($response['success']) {
            return [
                'success' => true,
                'refund_id' => $response['refund_id'],
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['error'],
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}
```

## 3D Secure / SCA Support

```php
function yourgateway_3ds_authenticate(array $params)
{
    // For Strong Customer Authentication (SCA)
    $challenge = YourGatewayAPI::initiateAuthentication([
        'amount' => $params['amount'],
        'currency' => $params['currency'],
        'return_url' => $params['returnurl'],
        'payment_data' => $params['card_token'],
    ]);
    
    return [
        'continue_redirect' => true,
        'redirect_url' => $challenge['redirect_url'],
    ];
}
```

## Step-by-Step Implementation

1. Create module directory in `/modules/gateways/yourgateway/`
2. Create main gateway file with config, link, and capture functions
3. Create callback handler for webhook processing
4. Implement signature verification
5. Create payment form template
6. Add logging for all transactions
7. Test in sandbox environment
8. Verify callback URL is accessible

## Real-World Examples

### Stripe Gateway Pattern

```php
function stripe_link($params) {
    \Stripe\Stripe::setApiKey($params['api_key']);
    
    $session = \Stripe\Checkout\Session::create([
        'payment_method_types' => ['card'],
        'line_items' => [[
            'price_data' => [
                'currency' => strtolower($params['currency']),
                'product_data' => ['name' => 'Invoice #' . $params['invoiceid']],
                'unit_amount' => $params['amount'] * 100,
            ],
            'quantity' => 1,
        ]],
        'mode' => 'payment',
        'success_url' => $params['returnurl'] . '&session_id={CHECKOUT_SESSION_ID}',
        'cancel_url' => $params['returnurl'] . '&canceled=true',
        'metadata' => ['invoice_id' => $params['invoiceid']],
    ]);
    
    return ['redirectUrl' => $session->url];
}
```

### PayPal Gateway Pattern

```php
function paypal_link($params) {
    $api = new PayPalAPI($params);
    
    $order = $api->createOrder([
        'amount' => $params['amount'],
        'currency' => $params['currency'],
        'description' => 'Invoice #' . $params['invoiceid'],
    ]);
    
    return [
        'rawPostData' => '',
        'type' => 'HTMLForm',
        'params' => [
            'action' => $api->getApprovalUrl($order),
            'method' => 'POST',
        ],
    ];
}
```

## Implementation Checklist

- [ ] Create module directory structure
- [ ] Implement MetaData function
- [ ] Implement config array
- [ ] Implement link function
- [ ] Create callback handler
- [ ] Verify webhook signatures
- [ ] Implement capture function (if needed)
- [ ] Implement refund function
- [ ] Create payment form template
- [ ] Add transaction logging
- [ ] Test in sandbox mode
- [ ] Configure webhook URL in gateway dashboard
- [ ] Enable IPN/signature verification
- [ ] Test end-to-end payment flow