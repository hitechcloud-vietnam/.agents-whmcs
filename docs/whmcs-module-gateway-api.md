# WHMCS Payment Gateway Module API

Complete reference for payment gateway module development in WHMCS.

## Overview

Payment gateway modules handle payment processing, refunds, and tokenization for WHMCS.

## Module Structure

### Required Files

```
modules/gateways/yourgateway/
├── yourgateway.php          # Main gateway file
└── callback.php              # Callback handler (optional)
```

### Basic Module Template

```php
<?php
/**
 * WHMCS Payment Gateway Module
 * 
 * Gateway: Your Gateway
 * Version: 1.0
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Define gateway metadata
 */
function yourgateway_MetaData()
{
    return [
        'DisplayName' => 'Your Gateway Name',
        'SystemName' => 'yourgateway',
        'APIVersion' => '1.0',
        'SupportsRecurring' => true,
        'Supports3DSecure' => true,
        'TokenisedInputsSupported' => true,
        'Description' => 'Description of your payment gateway',
    ];
}

/**
 * Define configuration options
 */
function yourgateway_ConfigArray()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Gateway',
        ],
        'APIKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'Your gateway API key',
        ],
        'APISecret' => [
            'FriendlyName' => 'API Secret',
            'Type' => 'password',
            'Size' => '50',
        ],
        'MerchantID' => [
            'FriendlyName' => 'Merchant ID',
            'Type' => 'text',
            'Size' => '30',
        ],
        'TestMode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
            'Description' => 'Enable test mode',
        ],
        'TransactionFee' => [
            'FriendlyName' => 'Transaction Fee (%)',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '0',
        ],
    ];
}
```

## Payment Functions

### link()

Generates payment button/links.

```php
/**
 * Link callback - generates payment link
 * 
 * @param array $params Gateway parameters
 * @return string HTML link
 */
function yourgateway_link(array $params)
{
    // Generate payment URL
    $testMode = !empty($params['config']['testmode']);
    $baseUrl = $testMode 
        ? 'https://sandbox.yourgateway.com'
        : 'https://api.yourgateway.com';
    
    $data = [
        'merchant_id' => $params['config']['merchantid'],
        'order_id' => $params['invoiceid'],
        'amount' => $params['amount'],
        'currency' => $params['currency'],
        'description' => 'Invoice #' . $params['invoicenum'],
        'return_url' => $params['returnurl'],
        'cancel_url' => $params['returnurl'],
        'notify_url' => $params['systemurl'] . '/modules/gateways/callback/yourgateway.php',
    ];
    
    // Generate signature
    $signature = generateSignature($data, $params['config']['apisecret']);
    $data['signature'] = $signature;
    
    // Build form
    $html = '<form action="' . $baseUrl . '/pay" method="POST">';
    foreach ($data as $key => $value) {
        $html .= '<input type="hidden" name="' . $key . '" value="' . htmlspecialchars($value) . '">';
    }
    $html .= '<input type="submit" value="Pay with Your Gateway">';
    $html .= '</form>';
    
    return $html;
}
```

### capture()

Captures payment (for payment buttons/forms).

```php
/**
 * Capture payment callback
 * 
 * @param array $params Gateway parameters
 * @return array Result
 */
function yourgateway_capture(array $params)
{
    try {
        // Initialize gateway
        $gateway = new YourGatewayAPI($params['config']);
        
        // Process payment
        $result = $gateway->charge([
            'amount' => $params['amount'],
            'currency' => $params['currency'],
            'card_token' => $_POST['token'],
            'description' => 'Invoice #' . $params['invoicenum'],
            'customer_email' => $params['clientdetails']['email'],
            'metadata' => [
                'invoice_id' => $params['invoiceid'],
                'whmcs_order_id' => $params['orderid'] ?? null,
            ],
        ]);
        
        if ($result['success']) {
            return [
                'success' => true,
                'transactionid' => $result['transaction_id'],
                'rawdata' => $result,
            ];
        }
        
        return ['error' => $result['message'] ?? 'Payment failed'];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### refund()

Processes a refund.

```php
/**
 * Refund callback
 * 
 * @param array $params Gateway parameters
 * @return array Result
 */
function yourgateway_refund(array $params)
{
    try {
        $gateway = new YourGatewayAPI($params['config']);
        
        $result = $gateway->refund([
            'transaction_id' => $params['transactionid'],
            'amount' => $params['amount'],
            'reason' => $params['reason'] ?? 'Customer request',
        ]);
        
        if ($result['success']) {
            return [
                'success' => true,
                'refundid' => $result['refund_id'],
                'rawdata' => $result,
            ];
        }
        
        return ['error' => $result['message'] ?? 'Refund failed'];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## Tokenization Functions

### create_subscription()

Creates a recurring billing subscription.

```php
/**
 * Create subscription callback
 * 
 * @param array $params Gateway parameters
 * @return array Result
 */
function yourgateway_create_subscription(array $params)
{
    try {
        $gateway = new YourGatewayAPI($params['config']);
        
        $result = $gateway->createSubscription([
            'customer_email' => $params['clientdetails']['email'],
            'card_token' => $_POST['token'],
            'amount' => $params['amount'],
            'interval' => mapBillingCycle($params['billingcycle']),
            'description' => 'Invoice #' . $params['invoicenum'],
        ]);
        
        if ($result['success']) {
            return [
                'success' => true,
                'subscriptionid' => $result['subscription_id'],
                'rawdata' => $result,
            ];
        }
        
        return ['error' => $result['message'] ?? 'Subscription creation failed'];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Map WHMCS billing cycle to gateway interval
 */
function mapBillingCycle(string $cycle): string
{
    $map = [
        'Monthly' => 'month',
        'Quarterly' => '3months',
        'Semi-Annually' => '6months',
        'Annually' => 'year',
        'Biennially' => '2years',
    ];
    
    return $map[$cycle] ?? 'month';
}
```

### remote_management()

Handles subscription management.

```php
/**
 * Remote management callback
 * 
 * @param array $params Gateway parameters
 * @return array Result
 */
function yourgateway_remote_management(array $params)
{
    try {
        $gateway = new YourGatewayAPI($params['config']);
        
        switch ($params['action']) {
            case 'cancel':
                $result = $gateway->cancelSubscription([
                    'subscription_id' => $params['subscriptionid'],
                ]);
                break;
                
            case 'update':
                $result = $gateway->updateSubscription([
                    'subscription_id' => $params['subscriptionid'],
                    'card_token' => $_POST['token'] ?? null,
                ]);
                break;
                
            case 'suspend':
                $result = $gateway->suspendSubscription([
                    'subscription_id' => $params['subscriptionid'],
                ]);
                break;
                
            case 'resume':
                $result = $gateway->resumeSubscription([
                    'subscription_id' => $params['subscriptionid'],
                ]);
                break;
                
            default:
                return ['error' => 'Unknown action'];
        }
        
        return ['success' => true, 'rawdata' => $result];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## 3D Secure Functions

### 3dsecure_redirect()

Handles 3D Secure authentication.

```php
/**
 * 3D Secure redirect callback
 * 
 * @param array $params Gateway parameters
 * @return array Result
 */
function yourgateway_3dsecure_redirect(array $params)
{
    try {
        $gateway = new YourGatewayAPI($params['config']);
        
        $result = $gateway->initiatePayment([
            'amount' => $params['amount'],
            'currency' => $params['currency'],
            'card_token' => $_POST['token'],
            '3d_required' => true,
            'return_url' => $params['systemurl'] . '/modules/gateways/callback/yourgateway.php?action=complete',
        ]);
        
        if (!empty($result['3d_url'])) {
            return [
                'success' => true,
                'rawdata' => $result,
            ];
        }
        
        // No 3D required, complete payment
        return capture($params);
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## API Client Example

```php
<?php
/**
 * Payment Gateway API Client
 */
class YourGatewayAPI
{
    private $config;
    private $apiUrl;
    
    public function __construct(array $config)
    {
        $this->config = $config;
        $this->apiUrl = !empty($config['testmode'])
            ? 'https://sandbox.yourgateway.com/v1'
            : 'https://api.yourgateway.com/v1';
    }
    
    public function charge(array $data): array
    {
        return $this->request('POST', '/charges', [
            'amount' => (int) ($data['amount'] * 100), // Convert to cents
            'currency' => strtolower($data['currency']),
            'source' => $data['card_token'],
            'description' => $data['description'],
            'receipt_email' => $data['customer_email'] ?? null,
            'metadata' => $data['metadata'] ?? [],
        ]);
    }
    
    public function refund(array $data): array
    {
        return $this->request('POST', '/refunds', [
            'charge' => $data['transaction_id'],
            'amount' => (int) ($data['amount'] * 100),
            'reason' => $data['reason'] ?? 'requested_by_customer',
        ]);
    }
    
    public function createSubscription(array $data): array
    {
        return $this->request('POST', '/subscriptions', [
            'customer_email' => $data['customer_email'],
            'source' => $data['card_token'],
            'plan' => $data['interval'],
            'amount' => (int) ($data['amount'] * 100),
            'description' => $data['description'],
        ]);
    }
    
    private function request(string $method, string $endpoint, array $data = []): array
    {
        $ch = curl_init();
        
        $url = $this->apiUrl . $endpoint;
        
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 30);
        curl_setopt($ch, CURLOPT_HTTPHEADER, [
            'Authorization: Bearer ' . $this->config['apikey'],
            'Content-Type: application/json',
        ]);
        
        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
        
        $response = curl_exec($ch);
        $error = curl_error($ch);
        curl_close($ch);
        
        if ($error) {
            throw new Exception("API Error: {$error}");
        }
        
        $result = json_decode($response, true);
        
        if (!$result['success'] && !empty($result['error'])) {
            throw new Exception($result['error']);
        }
        
        return $result;
    }
}
```

## Callback Handler

```php
<?php
/**
 * Gateway Callback Handler
 * modules/gateways/callback/yourgateway.php
 */

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../../../includes/gatewayfunctions.php';
require_once __DIR__ . '/../../../includes/functions.php';

checklangredirect();

// Verify callback authenticity
$gateway = Capsule::table('tblpaymentgateways')
    ->where('gateway', 'yourgateway')
    ->where('setting', 'value')
    ->pluck('value', 'setting')
    ->toArray();

// Parse webhook payload
$payload = file_get_contents('php://input');
$data = json_decode($payload, true);

// Verify signature
$signature = $_SERVER['HTTP_YOUR_SIGNATURE'];
$expected = hash_hmac('sha256', $payload, $gateway['apisecret']);

if (!hash_equals($expected, $signature)) {
    http_response_code(400);
    exit('Invalid signature');
}

// Process based on event type
switch ($data['event']) {
    case 'payment.completed':
        $transactionId = $data['transaction_id'];
        $invoiceId = $data['metadata']['invoice_id'];
        $amount = $data['amount'] / 100;
        
        addInvoicePayment($invoiceId, $transactionId, $amount, 'yourgateway');
        logTransaction('yourgateway', $data, 'Completed');
        break;
        
    case 'subscription.renewed':
        // Handle recurring billing
        break;
        
    case 'refund.created':
        logTransaction('yourgateway', $data, 'Refund');
        break;
}

http_response_code(200);
echo 'OK';
```

## Best Practices

1. **Always verify webhooks** - Check signatures before processing
2. **Idempotent operations** - Handle duplicate callbacks gracefully
3. **Use TLS** - Always use HTTPS for API calls
4. **Log transactions** - Log all gateway activity for debugging
5. **Handle failures** - Return appropriate HTTP status codes
6. **Tokenize cards** - Never store raw card numbers

## Related Documentation

- [whmcs-module-return-values.md](whmcs-module-return-values.md)
- [whmcs-module-error-handling.md](whmcs-module-error-handling.md)
- [whmcs-integration-payment.md](../integration/whmcs-integration-payment.md)