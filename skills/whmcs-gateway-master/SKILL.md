# WHMCS Gateway Master

## Overview
Master skill for WHMCS payment gateway module development. Covers payment processing, refund handling, webhook integration, and gateway best practices.

## Gateway Module Structure

```php
<?php
// /modules/gateways/yourgateway/yourgateway.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Gateway Module Functions
 */

function yourgateway_MetaData()
{
    return [
        'DisplayName' => 'Your Gateway Name',
        'APIVersion' => '1.1',
        'DisableLocalCreditCardInput' => false,
        'TokenisedStorage' => false,
        'LocalCreditCardInput' => true,
        'IndividualBillingIssues' => false,
    ];
}

function yourgateway_config(array $params = [])
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Gateway Name',
        ],
        'apiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'Your gateway API key',
        ],
        'apiSecret' => [
            'FriendlyName' => 'API Secret',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'Your gateway API secret',
        ],
        'merchantId' => [
            'FriendlyName' => 'Merchant ID',
            'Type' => 'text',
            'Size' => '30',
            'Description' => 'Your merchant account ID',
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
        'webhookSecret' => [
            'FriendlyName' => 'Webhook Secret',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'For webhook signature verification',
        ],
    ];
}

function yourgateway_link(array $params)
{
    $apiKey = $params['apiKey'];
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    $clientEmail = $params['clientdetails']['email'];
    $clientName = $params['clientdetails']['firstname'] . ' ' . $params['clientdetails']['lastname'];

    // Create payment session
    $session = createPaymentSession($params);

    $html = '<form method="post" action="' . $session['redirect_url'] . '">';
    $html .= '<input type="hidden" name="session_id" value="' . $session['id'] . '">';
    $html .= '<input type="hidden" name="amount" value="' . $amount . '">';
    $html .= '<input type="hidden" name="currency" value="' . $currency . '">';
    $html .= '<input type="hidden" name="invoice_id" value="' . $invoiceId . '">';
    $html .= '<input type="hidden" name="return_url" value="' . $params['returnurl'] . '">';
    $html .= '<input type="hidden" name="cancel_url" value="' . $params['cancelurl'] . '">';

    // Add card input fields
    $html .= '<div class="payment-form">';
    $html .= '<div class="form-group">';
    $html .= '<label>Card Number</label>';
    $html .= '<input type="text" name="card_number" class="form-control" placeholder="4242 4242 4242 4242" autocomplete="off">';
    $html .= '</div>';
    $html .= '<div class="form-row">';
    $html .= '<div class="form-group">';
    $html .= '<label>Expiry</label>';
    $html .= '<input type="text" name="card_expiry" class="form-control" placeholder="MM/YY">';
    $html .= '</div>';
    $html .= '<div class="form-group">';
    $html .= '<label>CVC</label>';
    $html .= '<input type="text" name="card_cvc" class="form-control" placeholder="123">';
    $html .= '</div>';
    $html .= '</div>';
    $html .= '</div>';

    $html .= '<button type="submit" class="btn btn-primary btn-block">';
    $html .= 'Pay ' . $amount . ' ' . $currency;
    $html .= '</button>';
    $html .= '</form>';

    // Add JavaScript for card validation
    $html .= '<script src="' . $params['systemurl'] . 'modules/gateways/yourgateway/validation.js"></script>';

    return $html;
}

/**
 * Refund Transaction
 */
function yourgateway_refund(array $params)
{
    $transactionId = $params['transid'];
    $amount = $params['amount'];
    $reason = $params['reason'] ?? '';

    $api = new YourGatewayAPI($params);

    try {
        $result = $api->refund([
            'transaction_id' => $transactionId,
            'amount' => $amount,
            'reason' => $reason,
        ]);

        if ($result['success']) {
            return [
                'status' => 'success',
                'refundid' => $result['refund_id'],
                'rawdata' => $result,
            ];
        }

        return [
            'status' => 'failed',
            'rawdata' => $result,
            'error' => $result['error_message'],
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'failed',
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Storn Transaction (Credit)
 */
function yourgateway_storn(array $params)
{
    // Similar to refund but with specific logic
    return yourgateway_refund($params);
}

/**
 * Capture Authorized Payment
 */
function yourgateway_capture(array $params)
{
    $transactionId = $params['transid'];

    $api = new YourGatewayAPI($params);

    try {
        $result = $api->capture([
            'transaction_id' => $transactionId,
        ]);

        if ($result['success']) {
            return [
                'status' => 'success',
                'transid' => $result['capture_id'],
                'rawdata' => $result,
            ];
        }

        return [
            'status' => 'failed',
            'error' => $result['error_message'],
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'failed',
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Void Authorized Payment
 */
function yourgateway_void(array $params)
{
    $transactionId = $params['transid'];

    $api = new YourGatewayAPI($params);

    try {
        $result = $api->void([
            'transaction_id' => $transactionId,
        ]);

        if ($result['success']) {
            return [
                'status' => 'success',
                'rawdata' => $result,
            ];
        }

        return [
            'status' => 'failed',
            'error' => $result['error_message'],
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'failed',
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Remote Input (for hosted gateways)
 */
function yourgateway_remote_input(array $params)
{
    $session = createPaymentSession($params);

    return [
        'inputType' => 'remote',
        'structure' => '<form action="' . $session['redirect_url'] . '" method="POST"></form>',
    ];
}

/**
 * 3D Secure Callback
 */
function yourgateway_secure_form(array $params)
{
    // Return iframe or redirect for 3DS authentication
    return [
        'type' => 'iframe',
        'url' => $params['three_d_secure_url'],
        'params' => [
            'PaReq' => $params['pareq'],
            'TermUrl' => $params['term_url'],
            'MD' => $params['md'],
        ],
    ];
}

/**
 * Token Creation
 */
function yourgateway_create_token(array $params)
{
    $api = new YourGatewayAPI($params);

    try {
        $result = $api->createCustomer([
            'email' => $params['client']['email'],
            'description' => 'WHMCS Client: ' . $params['client']['id'],
        ]);

        return [
            'success' => true,
            'token' => $result['customer_id'],
            'rawdata' => $result,
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Get Card Data (Tokenized)
 */
function yourgateway_get_arb(array $params)
{
    // Get stored card details
    $api = new YourGatewayAPI($params);

    try {
        $result = $api->getStoredCard([
            'customer_id' => $params['gatewayid'],
        ]);

        return [
            'status' => 'success',
            'cardData' => [
                'last_four' => $result['last4'],
                'expiry' => $result['exp_month'] . '/' . $result['exp_year'],
                'brand' => $result['brand'],
            ],
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'failed',
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Subscription Creation
 */
function yourgateway_subscription_create(array $params)
{
    $api = new YourGatewayAPI($params);

    try {
        $result = $api->createSubscription([
            'customer_id' => $params['gatewayid'],
            'amount' => $params['amount'],
            'currency' => $params['currency'],
            'interval' => $params['interval'],
            'description' => 'Invoice #' . $params['invoiceid'],
        ]);

        return [
            'status' => 'success',
            'subscriptionid' => $result['subscription_id'],
            'rawdata' => $result,
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'failed',
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Subscription Cancellation
 */
function yourgateway_subscription_cancel(array $params)
{
    $api = new YourGatewayAPI($params);

    try {
        $result = $api->cancelSubscription([
            'subscription_id' => $params['subscriptionid'],
        ]);

        return [
            'status' => 'success',
            'rawdata' => $result,
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'failed',
            'error' => $e->getMessage(),
        ];
    }
}
```

## API Client Class

```php
<?php
// /modules/gateways/yourgateway/lib/ApiClient.php

namespace WHMCS\Gateways\YourGateway;

class ApiClient
{
    private $apiKey;
    private $apiSecret;
    private $environment;
    private $webhookSecret;

    const SANDBOX_URL = 'https://sandbox.yourgateway.com/api';
    const PRODUCTION_URL = 'https://api.yourgateway.com/api';

    public function __construct(array $params)
    {
        $this->apiKey = $params['apiKey'] ?? '';
        $this->apiSecret = $params['apiSecret'] ?? '';
        $this->environment = $params['environment'] ?? 'sandbox';
        $this->webhookSecret = $params['webhookSecret'] ?? '';
    }

    private function getBaseUrl(): string
    {
        return $this->environment === 'production'
            ? self::PRODUCTION_URL
            : self::SANDBOX_URL;
    }

    private function signRequest(array $data): string
    {
        $payload = json_encode($data);
        $signature = hash_hmac('sha256', $payload, $this->apiSecret);
        return $signature;
    }

    public function request(string $method, string $endpoint, array $data = []): array
    {
        $url = $this->getBaseUrl() . $endpoint;
        $timestamp = time();

        $headers = [
            'Authorization: Bearer ' . $this->apiKey,
            'Content-Type: application/json',
            'X-Timestamp: ' . $timestamp,
            'X-Signature: ' . $this->signRequest(array_merge($data, ['timestamp' => $timestamp])),
        ];

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => $headers,
        ]);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        $result = json_decode($response, true);

        if ($httpCode >= 400) {
            throw new \Exception(
                $result['error']['message'] ?? 'API Error: HTTP ' . $httpCode
            );
        }

        return $result;
    }

    public function createPaymentIntent(array $params): array
    {
        return $this->request('POST', '/payments', [
            'amount' => (int)($params['amount'] * 100), // Convert to cents
            'currency' => strtolower($params['currency']),
            'metadata' => [
                'invoice_id' => $params['invoice_id'],
                'whmcs_gateway_id' => $params['gateway_id'],
            ],
        ]);
    }

    public function confirmPayment(string $paymentIntentId, array $cardData): array
    {
        return $this->request('POST', '/payments/' . $paymentIntentId . '/confirm', [
            'payment_method' => [
                'type' => 'card',
                'card' => [
                    'number' => $cardData['number'],
                    'exp_month' => $cardData['exp_month'],
                    'exp_year' => $cardData['exp_year'],
                    'cvc' => $cardData['cvc'],
                ],
            ],
        ]);
    }

    public function refund(array $params): array
    {
        return $this->request('POST', '/refunds', [
            'transaction_id' => $params['transaction_id'],
            'amount' => (int)($params['amount'] * 100),
            'reason' => $params['reason'] ?? 'customer_request',
        ]);
    }

    public function createCustomer(array $params): array
    {
        return $this->request('POST', '/customers', [
            'email' => $params['email'],
            'description' => $params['description'],
        ]);
    }

    public function createSubscription(array $params): array
    {
        return $this->request('POST', '/subscriptions', [
            'customer_id' => $params['customer_id'],
            'amount' => (int)($params['amount'] * 100),
            'currency' => strtolower($params['currency']),
            'interval' => $params['interval'],
            'description' => $params['description'],
        ]);
    }

    public function verifyWebhookSignature(string $payload, string $signature): bool
    {
        $expectedSignature = hash_hmac('sha256', $payload, $this->webhookSecret);
        return hash_equals($expectedSignature, $signature);
    }
}
```

## Webhook Handler

```php
<?php
// /modules/gateways/yourgateway/callback.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/ApiClient.php';

$gatewayParams = getGatewayParameters('yourgateway');

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $payload = file_get_contents('php://input');
    $signature = $_SERVER['HTTP_X_SIGNATURE'] ?? '';

    $api = new ApiClient($gatewayParams);

    // Verify webhook signature
    if (!$api->verifyWebhookSignature($payload, $signature)) {
        http_response_code(401);
        die('Invalid signature');
    }

    $event = json_decode($payload, true);

    logTransaction('yourgateway', $event, 'Webhook Received');

    switch ($event['type']) {
        case 'payment.success':
            handlePaymentSuccess($event);
            break;

        case 'payment.failed':
            handlePaymentFailed($event);
            break;

        case 'refund.created':
            handleRefundCreated($event);
            break;

        case 'charge.refunded':
            handleChargeRefunded($event);
            break;

        case 'subscription.payment_succeeded':
            handleSubscriptionPayment($event);
            break;

        case 'subscription.payment_failed':
            handleSubscriptionPaymentFailed($event);
            break;

        case 'subscription.cancelled':
            handleSubscriptionCancelled($event);
            break;

        default:
            logTransaction('yourgateway', $event, 'Unhandled Event Type');
    }

    http_response_code(200);
    echo 'OK';
}

function handlePaymentSuccess(array $event): void
{
    $payment = $event['data'];
    $invoiceId = $payment['metadata']['invoice_id'] ?? null;

    if ($invoiceId) {
        addInvoicePayment(
            $invoiceId,
            $payment['id'],
            $payment['amount'] / 100,
            0,
            'yourgateway'
        );

        logTransaction('yourgateway', $payment, 'Payment Success');
    }
}

function handlePaymentFailed(array $event): void
{
    $payment = $event['data'];
    $invoiceId = $payment['metadata']['invoice_id'] ?? null;

    if ($invoiceId) {
        logTransaction('yourgateway', $payment, 'Payment Failed');
    }
}

function handleRefundCreated(array $event): void
{
    $refund = $event['data'];

    logTransaction('yourgateway', $refund, 'Refund Created');
}

function handleChargeRefunded(array $event): void
{
    $refund = $event['data'];

    // Update transaction in WHMCS
    logTransaction('yourgateway', $refund, 'Charge Refunded');
}
```

## Configuration Table Setup

```sql
-- Gateway configuration stored in tblpaymentgateways
-- Example SQL for custom settings if needed

CREATE TABLE IF NOT EXISTS `mod_yourgateway_transactions` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `transaction_id` VARCHAR(255) NOT NULL UNIQUE,
    `whmcs_transaction_id` INT NULL,
    `invoice_id` INT NULL,
    `amount` DECIMAL(10,2) NOT NULL,
    `currency` VARCHAR(3) NOT NULL,
    `status` ENUM('pending', 'completed', 'failed', 'refunded') DEFAULT 'pending',
    `customer_id` VARCHAR(255) NULL,
    `subscription_id` VARCHAR(255) NULL,
    `metadata` JSON NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_invoice_id` (`invoice_id`),
    INDEX `idx_status` (`status`),
    INDEX `idx_customer_id` (`customer_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Best Practices

1. **Security First**: Always validate webhook signatures and use HTTPS
2. **Idempotency**: Handle duplicate webhook events gracefully
3. **Error Handling**: Log all errors and return appropriate HTTP status codes
4. **Transaction Logging**: Log all transactions for debugging and reconciliation
5. **Refund Validation**: Verify transaction ownership before processing refunds
6. **Currency Handling**: Always convert to smallest currency unit (cents)
7. **PCI Compliance**: Never store full card numbers; use tokenization
8. **Timeout Handling**: Set appropriate timeouts for API calls
9. **Retry Logic**: Implement retry logic with exponential backoff
10. **Testing**: Test all scenarios including edge cases and failures
