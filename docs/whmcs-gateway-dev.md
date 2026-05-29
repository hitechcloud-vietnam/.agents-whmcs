# WHMCS Payment Gateway Development

## Overview

Payment gateways process customer payments. WHMCS supports multiple gateway types including hosted, merchant, and third-party.

## Gateway Structure

```
modules/gateways/
├── yourgateway.php          # Main gateway file
├── callback/
│   └── yourgateway.php      # Callback handler
└── lang/
    └── english.php          # Language file
```

## Gateway Configuration

```php
<?php
function yourgateway_config(): array
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Credit Card',
        ],
        'merchantId' => [
            'FriendlyName' => 'Merchant ID',
            'Type' => 'text',
            'Size' => '40',
            'Description' => 'Your merchant ID from the payment provider',
        ],
        'apiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '40',
        ],
        'environment' => [
            'FriendlyName' => 'Environment',
            'Type' => 'dropdown',
            'Options' => 'Live,Test',
            'Default' => 'Test',
        ],
        'testMode' => [
            'FriendlyName' => 'Enable Test Mode',
            'Type' => 'yesno',
            'Description' => 'Enable test mode for development',
        ],
    ];
}
```

## Standard Gateway

### Link Function

```php
<?php
function yourgateway_link(array $params): string
{
    $merchantId = $params['merchantId'];
    $orderId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    $callbackUrl = $params['systemurl'] . '/modules/gateways/callback/yourgateway.php';
    $returnUrl = $params['returnurl'];
    
    // Generate unique transaction reference
    $txRef = 'INV-' . $orderId . '-' . time();
    
    // Store transaction reference
    $_SESSION['yourgateway_txref_' . $orderId] = $txRef;
    
    // Build payment form
    $html = '<form action="https://payments.provider.com/checkout" method="POST">';
    $html .= '<input type="hidden" name="merchant_id" value="' . htmlspecialchars($merchantId) . '">';
    $html .= '<input type="hidden" name="tx_ref" value="' . htmlspecialchars($txRef) . '">';
    $html .= '<input type="hidden" name="amount" value="' . $amount . '">';
    $html .= '<input type="hidden" name="currency" value="' . $currency . '">';
    $html .= '<input type="hidden" name="callback_url" value="' . htmlspecialchars($callbackUrl) . '">';
    $html .= '<input type="hidden" name="return_url" value="' . htmlspecialchars($returnUrl) . '">';
    $html .= '<input type="hidden" name="customer_email" value="' . htmlspecialchars($params['clientdetails']['email']) . '">';
    $html .= '<input type="hidden" name="description" value="Invoice #' . $orderId . '">';
    $html .= '<button type="submit" class="btn btn-primary">';
    $html .= 'Pay Now - ' . $params['currency'] . ' ' . number_format($amount, 2);
    $html .= '</button>';
    $html .= '</form>';
    
    return $html;
}
```

### Refund Function

```php
<?php
function yourgateway_refund(array $params): array
{
    try {
        $api = new PaymentGatewayApi($params['gateway']);
        
        $result = $api->refundPayment([
            'transaction_id' => $params['transid'],
            'amount' => $params['amount'],
            'reason' => 'Customer requested refund',
        ]);
        
        if ($result['success']) {
            return [
                'status' => 'success',
                'refund_id' => $result['refund_id'],
            ];
        }
        
        return [
            'status' => 'failed',
            'error' => $result['error'] ?? 'Refund failed',
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'failed',
            'error' => $e->getMessage(),
        ];
    }
}
```

## Merchant Gateway

```php
<?php
function yourgateway_config(): array
{
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'Credit Card (Direct)'],
        'merchantId' => ['FriendlyName' => 'Merchant ID', 'Type' => 'text'],
        'apiKey' => ['FriendlyName' => 'API Key', 'Type' => 'password'],
    ];
}

function yourgateway_capture(array $params): array
{
    $api = new MerchantApiClient($params);
    
    try {
        $result = $api->charge([
            'amount' => (int) ($params['amount'] * 100), // Convert to cents
            'currency' => strtolower($params['currency']),
            'card' => [
                'number' => $params['cardnum'],
                'exp_month' => $params['cardexp'],
                'exp_year' => $params['cardexpyear'],
                'cvv' => $params['cardcvv'],
            ],
            'description' => 'Invoice #' . $params['invoiceid'],
        ]);
        
        if ($result['success']) {
            return [
                'status' => 'success',
                'transid' => $result['transaction_id'],
                'rawdata' => $result,
            ];
        }
        
        return [
            'status' => 'declined',
            'rawdata' => $result,
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'error' => $e->getMessage(),
        ];
    }
}
```

## Callback Handler

```php
<?php
<?php
// modules/gateways/callback/yourgateway.php

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../../../includes/gatewayfunctions.php';
require_once __DIR__ . '/../../../includes/invoicefunctions.php';

define('GATEWAY_NAME', 'yourgateway');

$gatewayParams = getGatewayParameters('yourgateway');

// Get callback data
$webhookData = $_POST;

// Verify webhook signature
if (!verifyWebhookSignature($webhookData, $gatewayParams['apiKey'])) {
    logTransaction(GATEWAY_NAME, $webhookData, 'Invalid Signature');
    http_response_code(403);
    exit;
}

// Get stored transaction reference
$txRef = $webhookData['tx_ref'] ?? '';
$orderId = extractOrderId($txRef);

// Process based on status
$status = $webhookData['status'] ?? '';
$amount = $webhookData['amount'] ?? 0;

switch ($status) {
    case 'successful':
        $complete = completeOrder($orderId, $gatewayParams);
        
        if ($complete) {
            logTransaction(
                GATEWAY_NAME,
                $webhookData,
                'Successful',
                $amount
            );
        }
        break;
        
    case 'failed':
        logTransaction(
            GATEWAY_NAME,
            $webhookData,
            'Failed',
            $amount
        );
        break;
        
    case 'pending':
        logTransaction(
            GATEWAY_NAME,
            $webhookData,
            'Pending',
            $amount
        );
        break;
}

http_response_code(200);
exit;

function verifyWebhookSignature(array $data, string $apiKey): bool
{
    $signature = $data['signature'] ?? '';
    $payload = $data['tx_ref'] . $data['amount'] . $data['status'];
    
    $expectedSignature = hash_hmac('sha256', $payload, $apiKey);
    
    return hash_equals($expectedSignature, $signature);
}

function completeOrder(int $orderId, array $gatewayParams): bool
{
    $invoiceId = checkInvoiceInvoiceID($orderId);
    
    if (!$invoiceId) {
        return false;
    }
    
    addInvoicePayment(
        $invoiceId,
        $gatewayData['transaction_id'] ?? '',
        $gatewayData['amount'] ?? 0,
        0,
        GATEWAY_NAME
    );
    
    return true;
}

function extractOrderId(string $txRef): int
{
    // Extract order ID from tx_ref (e.g., INV-123-1234567890)
    $parts = explode('-', $txRef);
    return (int) ($parts[1] ?? 0);
}
```

## Remote Input Gateway

```php
<?php
function yourgateway_config(): array
{
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'PayPal Commerce'],
        'clientId' => ['FriendlyName' => 'Client ID', 'Type' => 'text'],
        'secret' => ['FriendlyName' => 'Secret', 'Type' => 'password'],
        'webhookId' => ['FriendlyName' => 'Webhook ID', 'Type' => 'text'],
    ];
}

function yourgateway_link(array $params): string
{
    $api = new PayPalApiClient($params);
    $order = $api->createOrder([
        'amount' => $params['amount'],
        'currency' => $params['currency'],
        'invoice_id' => $params['invoiceid'],
    ]);
    
    // Store PayPal order ID for callback
    $_SESSION['paypal_order_id'] = $order['id'];
    
    $jsUrl = $params['testMode'] === 'on' 
        ? 'https://www.sandbox.paypal.com/sdk/js' 
        : 'https://www.paypal.com/sdk/js';
    
    $html = '<script src="' . $jsUrl . '?client-id=' . $params['clientId'] . '"></script>';
    $html .= '<div id="paypal-button-container"></div>';
    $html .= '<script>
        paypal.Buttons({
            createOrder: function(data, actions) {
                return "' . $order['id'] . '";
            },
            onApprove: function(data, actions) {
                return actions.order.capture().then(function(details) {
                    window.location.href = "' . $params['returnurl'] . '&transaction_id=" + data.orderID;
                });
            }
        }).render("#paypal-button-container");
    </script>';
    
    return $html;
}
```

## Bank Transfer Gateway

```php
<?php
function yourbanktransfer_config(): array
{
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'Bank Transfer'],
        'bankName' => ['FriendlyName' => 'Bank Name', 'Type' => 'text'],
        'accountName' => ['FriendlyName' => 'Account Name', 'Type' => 'text'],
        'accountNumber' => ['FriendlyName' => 'Account Number', 'Type' => 'text'],
        'sortCode' => ['FriendlyName' => 'Sort Code', 'Type' => 'text'],
        'iban' => ['FriendlyName' => 'IBAN', 'Type' => 'text'],
        'swift' => ['FriendlyName' => 'SWIFT/BIC', 'Type' => 'text'],
        'instructions' => [
            'FriendlyName' => 'Payment Instructions',
            'Type' => 'textarea',
            'Rows' => '5',
        ],
    ];
}

function yourbanktransfer_link(array $params): string
{
    $html = '<div class="bank-transfer-details">';
    $html .= '<h4>Bank Transfer Payment</h4>';
    $html .= '<p>Please transfer the total amount to the following account:</p>';
    $html .= '<table class="table">';
    $html .= '<tr><td><strong>Bank:</strong></td><td>' . htmlspecialchars($params['bankName']) . '</td></tr>';
    $html .= '<tr><td><strong>Account Name:</strong></td><td>' . htmlspecialchars($params['accountName']) . '</td></tr>';
    $html .= '<tr><td><strong>Account Number:</strong></td><td>' . htmlspecialchars($params['accountNumber']) . '</td></tr>';
    $html .= '<tr><td><strong>Sort Code:</strong></td><td>' . htmlspecialchars($params['sortCode']) . '</td></tr>';
    if (!empty($params['iban'])) {
        $html .= '<tr><td><strong>IBAN:</strong></td><td>' . htmlspecialchars($params['iban']) . '</td></tr>';
    }
    if (!empty($params['swift'])) {
        $html .= '<tr><td><strong>SWIFT/BIC:</strong></td><td>' . htmlspecialchars($params['swift']) . '</td></tr>';
    }
    $html .= '<tr><td><strong>Amount:</strong></td><td>' . $params['currency'] . ' ' . number_format($params['amount'], 2) . '</td></tr>';
    $html .= '<tr><td><strong>Reference:</strong></td><td>INV-' . $params['invoiceid'] . '</td></tr>';
    $html .= '</table>';
    
    if (!empty($params['instructions'])) {
        $html .= '<div class="instructions">' . nl2br(htmlspecialchars($params['instructions'])) . '</div>';
    }
    
    $html .= '<p class="text-muted"><small>Please include your invoice number as payment reference.</small></p>';
    $html .= '</div>';
    
    return $html;
}
```

## Best Practices

1. **Always verify webhooks** - Validate signatures before processing
2. **Use transactions** - Log all payment activities
3. **Handle idempotency** - Prevent duplicate processing
4. **Return proper status** - Use correct return format
5. **Secure credentials** - Never log sensitive card data

## Related Documentation

- [WHMCS Payment Hooks](/docs/whmcs-payment-hooks.md)