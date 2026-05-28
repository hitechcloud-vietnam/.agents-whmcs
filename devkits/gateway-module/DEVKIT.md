# WHMCS Payment Gateway DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/gateway-module/
├── standard.php          # Standard redirect gateway
├── merchant.php          # Merchant/capture gateway
├── tokenization.php      # Tokenization gateway
├── remote-input.php      # Remote input gateway
├── callback.php         # Callback handler template
└── DEVKIT.md            # This file
```

## Standard Redirect Gateway

```php
<?php
/**
 * WHMCS Payment Gateway: {gateway}
 * Standard Redirect Gateway Template
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {gateway}_config(): array {
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => '{Display Name}',
        ],
        'description' => [
            'Type' => 'System',
            'Value' => 'Payment gateway description',
        ],
        'merchantId' => [
            'FriendlyName' => 'Merchant ID',
            'Type' => 'text',
            'Size' => '30',
        ],
        'apiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
        ],
        'testMode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
        ],
    ];
}

function {gateway}_link(array $params): string {
    $orderId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    $returnUrl = $params['returnurl'];
    $clientName = $params['clientname'];

    // Build payment request
    $paymentData = [
        'merchant_id' => $params['merchantId'],
        'order_id' => $orderId,
        'amount' => $amount,
        'currency' => $currency,
        'return_url' => $returnUrl,
        'customer_name' => $clientName,
        'timestamp' => time(),
    ];

    // Generate signature
    $signature = generateSignature($paymentData, $params['apiKey']);

    $endpoint = $params['testMode'] === 'on'
        ? 'https://sandbox.{provider}.com/payment'
        : 'https://api.{provider}.com/payment';

    $html = '<form action="' . $endpoint . '" method="POST" id="{gateway}_form">';
    $html .= '<input type="hidden" name="merchant_id" value="' . htmlspecialchars($paymentData['merchant_id']) . '">';
    $html .= '<input type="hidden" name="order_id" value="' . $orderId . '">';
    $html .= '<input type="hidden" name="amount" value="' . $amount . '">';
    $html .= '<input type="hidden" name="currency" value="' . htmlspecialchars($currency) . '">';
    $html .= '<input type="hidden" name="return_url" value="' . htmlspecialchars($returnUrl) . '">';
    $html .= '<input type="hidden" name="signature" value="' . htmlspecialchars($signature) . '">';
    $html .= '<input type="submit" value="Pay Now" class="btn btn-primary">';
    $html .= '</form>';
    $html .= '<script>document.getElementById("{gateway}_form").submit();</script>';

    return $html;
}

function {gateway}_refund(array $params): array {
    $transactionId = $params['transid'];
    $amount = $params['amount'];

    try {
        $api = new \{Gateway}\ApiClient($params);
        $result = $api->refund($transactionId, $amount);

        logTransaction('{Gateway}', $params, 'Refund', $result);

        return [
            'status' => 'success',
            'transid' => $result['refund_id'],
            'raw' => $result,
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'failed',
            'error' => $e->getMessage(),
        ];
    }
}

function generateSignature(array $data, string $apiKey): string {
    ksort($data);
    $stringToSign = [];
    foreach ($data as $key => $value) {
        if ($key !== 'signature') {
            $stringToSign[] = "$key=$value";
        }
    }
    $signature = implode('&', $stringToSign) . '&key=' . $apiKey;

    return hash_hmac('sha256', $signature, $apiKey);
}
```

## Merchant Gateway (Card Capture)

```php
<?php
/**
 * WHMCS Merchant Gateway: {gateway}
 * Card Capture Gateway Template
 */

function {gateway}_config(): array {
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => '{Display Name}',
        ],
        'merchantId' => [
            'FriendlyName' => 'Merchant ID',
            'Type' => 'text',
        ],
        'apiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
        ],
        'testMode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
        ],
    ];
}

function {gateway}_capture_form(array $params): string {
    return '<form>
        <div class="form-group">
            <label>Card Number</label>
            <input type="text" name="card_number" class="form-control" 
                   placeholder="1234 5678 9012 3456" maxlength="19">
        </div>
        <div class="form-row">
            <div class="form-group">
                <label>Expiry</label>
                <input type="text" name="expiry" class="form-control" 
                       placeholder="MM/YY" maxlength="5">
            </div>
            <div class="form-group">
                <label>CVV</label>
                <input type="text" name="cvv" class="form-control" 
                       placeholder="123" maxlength="4">
            </div>
        </div>
        <input type="submit" value="Pay $" . $params['amount'] . ' class="btn btn-primary">
    </form>';
}

function {gateway}_capture(array $params): array {
    $cardData = [
        'number' => $params['card_number'],
        'expiry' => $params['expiry'],
        'cvv' => $params['cvv'],
    ];

    try {
        $api = new \{Gateway}\ApiClient($params);
        $result = $api->charge($params['invoiceid'], $params['amount'], $cardData);

        logTransaction('{Gateway}', [
            'invoice' => $params['invoiceid'],
            'amount' => $params['amount'],
            'card_last4' => substr($cardData['number'], -4),
        ], 'Charge', $result);

        return [
            'status' => 'success',
            'transid' => $result['transaction_id'],
            'raw' => $result,
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'failed',
            'error' => $e->getMessage(),
        ];
    }
}

function {gateway}_refund(array $params): array {
    try {
        $api = new \{Gateway}\ApiClient($params);
        $result = $api->refund($params['transid'], $params['amount']);

        return [
            'status' => 'success',
            'transid' => $result['refund_id'],
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'failed',
            'error' => $e->getMessage(),
        ];
    }
}
```

## Callback Handler Template

```php
<?php
/**
 * WHMCS Payment Gateway Callback: {gateway}
 * IPN/Callback Handler
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {gateway}_callback(): void {
    $gateway = $_POST;

    // Log raw callback for debugging
    logTransaction('{Gateway}', $gateway, 'Callback Received');

    // Validate callback
    if (!validateCallback($gateway)) {
        logTransaction('{Gateway}', $gateway, 'Invalid Signature');
        die('Invalid callback');
    }

    // Get params from gateway settings
    $params = getGatewayParams('{gateway}');

    $transactionId = $gateway['transaction_id'] ?? '';
    $orderId = $gateway['order_id'] ?? '';
    $amount = $gateway['amount'] ?? 0;
    $status = $gateway['status'] ?? '';
    $signature = $gateway['signature'] ?? '';

    // Map status to WHMCS status
    $WHMCSStatus = match ($status) {
        'success' => 'Success',
        'pending' => 'Pending',
        'failed' => 'Failed',
        default => 'Pending',
    };

    // Log and store transaction
    logTransaction('{Gateway}', $gateway, $WHMCSStatus);

    // If successful, add invoice payment
    if ($status === 'success') {
        addInvoicePayment(
            $orderId,
            $transactionId,
            $amount,
            0,
            '{Gateway}'
        );
    }

    // Redirect back to WHMCS
    $systemUrl = \WHMCS\Config\Setting::getValue('SystemURL');
    header('Location: ' . $systemUrl . '/viewinvoice.php?id=' . $orderId);
    exit;
}

function validateCallback(array $gateway): bool {
    $params = getGatewayParams('{gateway}');

    // Rebuild signature for verification
    $signature = generateSignature($gateway, $params['apiKey']);

    return hash_equals($signature, $gateway['signature'] ?? '');
}

function generateSignature(array $data, string $apiKey): string {
    ksort($data);
    $stringToSign = [];
    foreach ($data as $key => $value) {
        if ($key !== 'signature' && $key !== '') {
            $stringToSign[] = "$key=$value";
        }
    }
    $signature = implode('&', $stringToSign) . '&key=' . $apiKey;

    return hash_hmac('sha256', $signature, $apiKey);
}
```

## Checklist

```
Pre-Dev:
□ Get gateway API documentation
□ Identify gateway type (standard/merchant/token/remote)
□ Get sandbox/test credentials
□ Understand signature validation method

Development:
□ Implement config() with all settings
□ Implement link() → Return HTML form (standard)
□ Or implement capture_form() + capture() (merchant)
□ Create callback handler with signature validation
□ Add IP whitelist check
□ Implement refund() for refunds

Security:
□ Validate signature on every callback
□ IP whitelist for callback source
□ Sanitize all received data
□ Log all transactions
□ Never log full card numbers

Testing:
□ Test with sandbox credentials
□ Test successful payment flow
□ Test failed payment handling
□ Test refund processing
□ Verify callback IP whitelist
□ Test duplicate callback handling
```
