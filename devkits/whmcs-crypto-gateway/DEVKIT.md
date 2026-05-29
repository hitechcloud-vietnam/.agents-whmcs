# WHMCS Crypto Gateway Module - DEVKIT

## Module Information
- **Name**: Crypto Gateway
- **Version**: 1.0.0
- **Type**: Gateway Module
- **Description**: Cryptocurrency payment gateway supporting BTC, ETH, and more

## Installation
1. Copy to `/modules/gateways/crypto/`
2. Activate via WHMCS Admin > Configuration > Payment Gateways

## crypto.php
```php
<?php
/**
 * WHMCS Cryptocurrency Payment Gateway
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function crypto_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Cryptocurrency'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'Cryptocurrency payment gateway (BTC, ETH, etc.)'
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '80',
            'Description' => 'Crypto payment processor API key'
        ],
        'webhook_secret' => [
            'FriendlyName' => 'Webhook Secret',
            'Type' => 'password',
            'Size' => '80',
            'Description' => 'Webhook verification secret'
        ],
        'supported_coins' => [
            'FriendlyName' => 'Supported Coins',
            'Type' => 'text',
            'Size' => '100',
            'Default' => 'BTC,ETH,USDT',
            'Description' => 'Comma-separated list of supported cryptocurrencies'
        ],
        'network_fee_percent' => [
            'FriendlyName' => 'Network Fee (%)',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '0',
            'Description' => 'Percentage added to cover network fees'
        ],
        'confirmation_required' => [
            'FriendlyName' => 'Confirmations Required',
            'Type' => 'dropdown',
            'Options' => [
                '1' => '1 (Fast)',
                '3' => '3 (Standard)',
                '6' => '6 (High Security)'
            ],
            'Default' => '3'
        ],
        'qr_code_logo' => [
            'FriendlyName' => 'QR Code Logo URL',
            'Type' => 'text',
            'Size' => '100',
            'Description' => 'URL to logo for QR codes'
        ]
    ];
}

function crypto_capture($params)
{
    $apiKey = $params['api_key'];
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    $supportedCoins = explode(',', $params['supported_coins']);
    $clientEmail = $params['clientdetails']['email'] ?? '';
    
    // Create payment invoice with crypto processor
    $payload = [
        'order_id' => 'INV_' . $invoiceId . '_' . time(),
        'amount' => $amount,
        'currency' => $currency,
        'email' => $clientEmail,
        'coins' => $supportedCoins,
        'callback_url' => $params['systemurl'] . '/modules/gateways/callback/crypto.php',
        'success_url' => $params['returnurl'],
        'cancel_url' => $params['cancelurl']
    ];
    
    $ch = curl_init('https://api.crypto-processor.com/v1/invoices');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json',
        'Authorization: Bearer ' . $apiKey
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['id']) && isset($result['payment_addresses'])) {
        return [
            'status' => 'pending',
            'declined' => false,
            'pending' => true,
            'rawsuccess' => true,
            'payment_addresses' => $result['payment_addresses'],
            'invoice_url' => $result['invoice_url'] ?? '',
            'reference' => $result['id']
        ];
    } else {
        return [
            'status' => 'failed',
            'error' => $result['error'] ?? 'Failed to create invoice'
        ];
    }
}

function crypto_callback($params)
{
    $webhookSecret = $params['webhook_secret'];
    
    // Get webhook data
    $payload = file_get_contents('php://input');
    $signature = $_SERVER['HTTP_X_SIGNATURE'] ?? '';
    
    // Verify webhook signature
    $expectedSig = hash_hmac('sha256', $payload, $webhookSecret);
    
    if ($signature !== $expectedSig) {
        return ['status' => 'error', 'rawdata' => 'Invalid signature'];
    }
    
    $data = json_decode($payload, true);
    
    if ($data['event'] == 'invoice.paid') {
        $invoiceData = $data['data'];
        $invoiceId = str_replace('INV_', '', $invoiceData['order_id']);
        $invoiceId = explode('_', $invoiceId)[0];
        
        return [
            'status' => 'success',
            'transid' => $invoiceData['id'],
            'amount' => $invoiceData['amount_paid'],
            'currency' => $invoiceData['currency'],
            'confirmations' => $invoiceData['confirmations'],
            'rawdata' => json_encode($data)
        ];
    }
    
    if ($data['event'] == 'invoice.underpaid') {
        return [
            'status' => 'pending',
            'rawdata' => json_encode($data)
        ];
    }
    
    return ['status' => 'pending'];
}

function crypto_refund($params)
{
    $apiKey = $params['api_key'];
    $transactionId = $params['transactionId'];
    $amount = $params['amount'];
    $cryptoAddress = $params['crypto_address'] ?? '';
    $currency = $params['currency'];
    
    if (empty($cryptoAddress)) {
        return ['status' => 'failed', 'error' => 'No crypto address provided for refund'];
    }
    
    $payload = [
        'transaction_id' => $transactionId,
        'amount' => $amount,
        'currency' => $currency,
        'address' => $cryptoAddress
    ];
    
    $ch = curl_init('https://api.crypto-processor.com/v1/refunds');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json',
        'Authorization: Bearer ' . $apiKey
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['id'])) {
        return [
            'status' => 'success',
            'refund_id' => $result['id'],
            'tx_hash' => $result['tx_hash'] ?? ''
        ];
    } else {
        return [
            'status' => 'failed',
            'error' => $result['error'] ?? 'Refund failed'
        ];
    }
}

function crypto_query($params)
{
    $apiKey = $params['api_key'];
    $transactionId = $params['reference'];
    
    $ch = curl_init('https://api.crypto-processor.com/v1/invoices/' . $transactionId);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiKey
    ]);
    
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
    if ($vars['paymentmethod'] == 'crypto') {
        logActivity("Crypto payment confirmed for Invoice #" . $vars['invoiceid']);
        
        // Send confirmation email
        $client = full_query("SELECT email FROM " . TABLE_PREFIX . "tblclients WHERE id = " . (int)$vars['userid']);
        $emailData = mysql_fetch_array($client);
        
        if ($emailData) {
            send_email(
                $emailData['email'],
                'crypto_payment_confirmation',
                [
                    'invoice_id' => $vars['invoiceid'],
                    'transid' => $vars['transid'],
                    'amount' => $vars['amount']
                ]
            );
        }
    }
});

add_hook('DailyCronJob', 1, function($vars) {
    CryptoHelper::checkPendingTransactions();
});

class CryptoHelper
{
    public static function checkPendingTransactions()
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . "tblaccounts
            WHERE paymentmethod = 'crypto'
            AND transid IN (
                SELECT reference FROM " . TABLE_PREFIX . "mod_crypto_pending
            )
        ");
        
        while ($payment = mysql_fetch_array($result)) {
            // Check if payment is confirmed
            $status = CryptoHelper::getTransactionStatus($payment['transid']);
            
            if ($status['confirmations'] >= 3) {
                // Mark as complete
            }
        }
    }
    
    public static function getTransactionStatus($transactionId)
    {
        return ['confirmations' => 0, 'status' => 'pending'];
    }
}
```