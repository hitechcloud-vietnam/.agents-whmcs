# WHMCS Payment Gateway Setup

## Concept Explanation

Payment gateways process customer payments for WHMCS invoices. WHMCS supports numerous built-in gateways and allows custom module development. Proper gateway configuration ensures reliable payment collection, supports multiple currencies, handles failed payments gracefully, and maintains PCI compliance.

### Gateway Types

- **Hosted Gateways**: Redirect to third-party payment pages (PayPal, Stripe)
- **Direct Gateways**: Process payments within WHMCS (credit card on file)
- **Tokenized Gateways**: Store tokens for recurring charges
- **Aggregator Gateways**: Consolidate multiple payment methods
- **Cryptocurrency Gateways**: Accept crypto payments

## Code Patterns & Templates

### Gateway Module Structure

```php
<?php
// modules/gateways/mygateway/mygateway.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Define gateway module configuration
 */
function mygateway_MetaData() {
    return [
        'DisplayName' => 'My Custom Gateway',
        'APIVersion' => '1.0',
        'DisableLocalCredtCardInput' => false,
        'TokenisedStorage' => true,
    ];
}

/**
 * Define configuration options
 */
function mygateway_config() {
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'My Custom Gateway'
        ],
        'apiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'Your gateway API key'
        ],
        'apiSecret' => [
            'FriendlyName' => 'API Secret',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'Your gateway API secret'
        ],
        'environment' => [
            'FriendlyName' => 'Environment',
            'Type' => 'dropdown',
            'Options' => [
                'live' => 'Production',
                'test' => 'Sandbox/Test'
            ],
            'Default' => 'test'
        ],
        'webhookSecret' => [
            'FriendlyName' => 'Webhook Secret',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'For verifying webhook signatures'
        ]
    ];
}

/**
 * Link to external payment page
 */
function mygateway_link($params) {
    $apiKey = $params['apiKey'];
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    $email = $params['clientdetails']['email'];
    
    // Create payment session with gateway
    $sessionData = [
        'amount' => $amount,
        'currency' => $currency,
        'description' => 'Invoice #' . $params['invoicenum'],
        'metadata' => [
            'whmcs_invoice_id' => $invoiceId,
            'whmcs_client_id' => $params['clientdetails']['userid']
        ]
    ];
    
    // Create checkout session
    $session = createGatewaySession($apiKey, $sessionData);
    
    $html = '<form action="' . $session['url'] . '" method="POST">';
    $html .= '<input type="hidden" name="session_id" value="' . $session['id'] . '">';
    $html .= '<input type="submit" value="' . $params['langpaynow'] . '">';
    $html .= '</form>';
    
    return $html;
}

/**
 * Handle payment callback (IPN/Webhook)
 */
function mygateway_callback($params) {
    $payload = file_get_contents('php://input');
    $signature = $_SERVER['HTTP_X_SIGNATURE'] ?? '';
    
    // Verify webhook signature
    if (!verifyWebhookSignature($payload, $signature, $params['webhookSecret'])) {
        logTransaction($params['gateway'], $_REQUEST, 'Invalid Signature');
        return ['error' => 'Invalid signature'];
    }
    
    $data = json_decode($payload, true);
    $invoiceId = $data['metadata']['whmcs_invoice_id'];
    $transactionId = $data['id'];
    $amount = $data['amount'];
    $status = $data['status'];
    
    if ($status === 'succeeded') {
        addInvoicePayment($invoiceId, $transactionId, $amount, 0, 'mygateway');
        logTransaction($params['gateway'], $data, 'Successful');
    } elseif ($status === 'pending') {
        logTransaction($params['gateway'], $data, 'Pending');
    } else {
        logTransaction($params['gateway'], $data, 'Failed');
    }
    
    return ['success' => true];
}

/**
 * Refund a transaction
 */
function mygateway_refund($params) {
    $transactionId = $params['transid'];
    $amount = $params['amount'];
    $apiKey = $params['apiKey'];
    
    $result = processRefund($apiKey, $transactionId, $amount);
    
    if ($result['success']) {
        return ['status' => 'success', 'rawdata' => $result];
    } else {
        return ['status' => 'error', 'rawdata' => $result];
    }
}

/**
 * Storing token for recurring billing
 */
function mygateway_storeremote($params) {
    $cardNumber = decrypt($params['ccinfo']['cardnum']);
    $expiryMonth = $params['ccinfo']['cardexp'];
    $expiryYear = $params['ccinfo']['cardexp2'];
    
    // Tokenize card with payment provider
    $tokenData = [
        'card_number' => $cardNumber,
        'expiry_month' => substr($expiryMonth, 0, 2),
        'expiry_year' => '20' . $expiryYear,
        'cvv' => $params['ccinfo']['cccvv']
    ];
    
    $token = createCardToken($params['apiKey'], $tokenData);
    
    return [
        'status' => 'success',
        'remote_token' => $token['id'],
        'card_last_four' => substr($cardNumber, -4),
        'card_type' => detectCardType($cardNumber)
    ];
}
```

### Gateway Configuration Hook

```php
<?php
// hooks/gateway_config.php

// Set default gateway based on client country
add_hook('ClientAreaPrimarySidebar', 1, function($vars) {
    $client = $_SESSION['uid'] ? getClientsDetails($_SESSION['uid']) : null;
    
    if ($client) {
        // Map countries to preferred gateways
        $gatewayMap = [
            'US' => 'stripe',
            'CA' => 'stripe',
            'GB' => 'paypal',
            'DE' => 'paypal',
            'AU' => 'stripe',
            'default' => 'paypal'
        ];
        
        $preferredGateway = $gatewayMap[$client['country']] ?? $gatewayMap['default'];
        
        $_SESSION['preferred_gateway'] = $preferredGateway;
    }
});

// Auto-select payment method based on amount
add_hook('InvoicePageLoad', 1, function($vars) {
    $invoice = $vars['invoice'];
    
    // Force PayPal for large transactions (fraud prevention)
    if ($invoice['total'] > 1000) {
        return ['forced_gateway' => 'paypal'];
    }
    
    // Prefer Stripe for smaller transactions
    if ($invoice['total'] < 100) {
        return ['forced_gateway' => 'stripe'];
    }
});
```

### Gateway Fallback System

```php
<?php
// includes/gateway_fallback.php

class GatewayFallback {
    
    /**
     * Get available gateways with fallback
     */
    public static function getAvailableGateways($invoiceId) {
        $gateways = [];
        
        $primaryGateway = self::getPrimaryGateway($invoiceId);
        $fallbackGateways = self::getFallbackGateways();
        
        // Check if primary is available
        if (self::isGatewayAvailable($primaryGateway)) {
            $gateways[] = $primaryGateway;
        } else {
            // Fall back to alternatives
            foreach ($fallbackGateways as $gateway) {
                if (self::isGatewayAvailable($gateway)) {
                    $gateways[] = $gateway;
                    break;
                }
            }
        }
        
        // Add all available gateways
        foreach (getActivePaymentGateways() as $gw) {
            if (!in_array($gw['sysname'], $gateways)) {
                $gateways[] = $gw['sysname'];
            }
        }
        
        return $gateways;
    }
    
    /**
     * Process payment with automatic gateway switching
     */
    public static function processWithFallback($invoiceId, $cardData) {
        $gateways = self::getAvailableGateways($invoiceId);
        
        foreach ($gateways as $gateway) {
            $result = callGatewayFunction($gateway, 'link', [
                'invoiceid' => $invoiceId,
                'card_data' => $cardData
            ]);
            
            if ($result['success']) {
                return $result;
            }
            
            logActivity("Gateway $gateway failed, trying next...", null);
        }
        
        return ['success' => false, 'error' => 'All payment methods failed'];
    }
}
```

## Step-by-Step Implementation

### 1. Choose Gateway Type

Determine which gateway type suits your needs:
- Hosted (redirect) for PCI compliance
- Direct for seamless experience
- Tokenized for subscriptions

### 2. Create Gateway Module

```bash
mkdir -p modules/gateways/mygateway/
touch modules/gateways/mygateway/mygateway.php
```

### 3. Implement Required Functions

- `gateway_MetaData()` - Module metadata
- `gateway_config()` - Configuration options
- `gateway_link()` - Payment page redirect
- `gateway_callback()` - IPN handling

### 4. Add Optional Functions

- `gateway_refund()` - Refund processing
- `gateway_storeremote()` - Tokenization
- `gateway_capture()` - Capture authorized payments
- `gateway3dsecure()` - 3D Secure handling

### 5. Test Gateway

1. Enable in WHMCS admin
2. Configure with test credentials
3. Create test invoice
4. Process test payment
5. Verify callback handling

## Examples

### Example: Custom PayPal Integration

```php
<?php
function paypalcustom_link($params) {
    $paypalEmail = $params['PayPalEmail'];
    $invoiceNum = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    
    $url = 'https://www.paypal.com/cgi-bin/webscr?cmd=_xclick';
    $url .= '&business=' . urlencode($paypalEmail);
    $url .= '&item_name=' . urlencode('Invoice #' . $params['invoicenum']);
    $url .= '&invoice=' . $invoiceNum;
    $url .= '&amount=' . $amount;
    $url .= '&currency_code=' . $currency;
    $url .= '&return=' . urlencode($params['systemurl'] . '/viewinvoice.php?id=' . $invoiceNum);
    $url .= '&notify_url=' . urlencode($params['systemurl'] . '/modules/gateways/paypal.php');
    
    return '<form action="' . $url . '" method="post">
        <input type="image" src="https://www.paypalobjects.com/webstatic/en_US/i/btn/png/btn_paynow_ CC0.png" 
            border="0" name="submit" alt="PayPal">
    </form>';
}
```

### Example: Stripe Credit Card Processing

```php
<?php
function stripe_link($params) {
    return '<script src="https://js.stripe.com/v3/"></script>
    <form id="payment-form">
        <div id="card-element"></div>
        <button id="submit">Pay ' . formatCurrency($params['amount']) . '</button>
    </form>
    <script>
        const stripe = Stripe("' . $params['publishableKey'] . '");
        const elements = stripe.elements();
        const card = elements.create("card");
        card.mount("#card-element");
        
        document.getElementById("payment-form").addEventListener("submit", async (e) => {
            e.preventDefault();
            const {paymentIntent, error} = await stripe.confirmCardPayment(
                "' . $params['client_secret'] . '",
                {payment_method: {card: card}}
            );
            if (error) {
                alert(error.message);
            } else if (paymentIntent.status === "succeeded") {
                window.location.href = "' . $params['systemurl'] . '/cleintarea.php?success=1";
            }
        });
    </script>';
}
```

## Implementation Checklist

- [ ] Choose gateway type for business needs
- [ ] Create gateway module directory
- [ ] Implement required module functions
- [ ] Add configuration options
- [ ] Implement webhook/callback handling
- [ ] Add refund capability
- [ ] Test in sandbox environment
- [ ] Verify PCI compliance requirements
- [ ] Configure in WHMCS admin
- [ ] Test end-to-end payment flow
- [ ] Set up monitoring for failed payments
- [ ] Document gateway usage procedures
