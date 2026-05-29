# WHMCS Cryptocurrency Payment Setup Workflow

## Description
Configure cryptocurrency payments (Bitcoin, Ethereum, etc.) for WHMCS.

## Prerequisites
- Cryptocurrency payment processor (Coinbase Commerce, BitPay, etc.)
- Business account with processor
- WHMCS 7.0+

## Steps

### Step 1: Get Cryptocurrency Gateway Setup
```bash
# Choose a cryptocurrency payment processor:
# - Coinbase Commerce
# - BitPay
# - CoinPayments
# - BTCPay Server (self-hosted)
```

### Step 2: Create Crypto Gateway
```php
<?php
// modules/gateways/crypto/crypto.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function crypto_MetaData()
{
    return [
        'DisplayName' => 'Cryptocurrency',
        'APIVersion' => '1.0',
    ];
}

function crypto_config()
{
    return [
        'FriendlyName' => ['value' => 'Cryptocurrency Payment'],
        'apiKey' => ['Type' => 'password', 'Label' => 'API Key'],
        'webhookSecret' => ['Type' => 'password', 'Label' => 'Webhook Secret'],
        'supportedCoins' => [
            'Type' => 'text',
            'Label' => 'Supported Coins',
            'Description' => 'Comma-separated: BTC,ETH,LTC',
            'Default' => 'BTC,ETH',
        ],
        'environment' => ['Type' => 'dropdown', 'Options' => 'test,live'],
    ];
}

function crypto_link($params)
{
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    $coins = explode(',', $params['supportedCoins']);
    
    // Create crypto invoice with processor
    $invoice = createCryptoInvoice([
        'amount' => $amount,
        'currency' => $currency,
        'metadata' => [
            'invoice_id' => $invoiceId,
            'customer_email' => $params['clientdetails']['email'],
        ],
    ]);
    
    // Store charge ID
    $_SESSION['crypto_charge_id'] = $invoice['id'];
    
    $html = '<div class="crypto-payment">';
    $html .= '<h4>Pay with Cryptocurrency</h4>';
    $html .= '<p>Amount: ' . formatCurrency($amount) . '</p>';
    $html .= '<p>Select cryptocurrency:</p>';
    
    $html .= '<div class="crypto-options">';
    foreach ($coins as $coin) {
        $coin = trim($coin);
        $address = $invoice['addresses'][$coin] ?? '';
        $qrCode = generateQRCode($address, $amount);
        
        $html .= '<div class="crypto-option" data-coin="' . $coin . '">';
        $html .= '<img src="' . $qrCode . '" alt="' . $coin . ' QR">';
        $html .= '<strong>' . strtoupper($coin) . '</strong>';
        $html .= '<code>' . $address . '</code>';
        $html .= '<button onclick="copyAddress(\'' . $address . '\')">Copy Address</button>';
        $html .= '</div>';
    }
    $html .= '</div>';
    
    $html .= '<p class="crypto-timer">Time remaining: <span id="crypto-countdown">30:00</span></p>';
    $html .= '</div>';
    
    return $html;
}
```

### Step 3: Handle Crypto Webhooks
```php
<?php
// modules/gateways/callback/crypto/webhook.php

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../../../includes/gatewayfunctions.php';

$gateway = getGatewayVariables('crypto');

// Verify webhook signature
$payload = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'];

if (!verifyCryptoSignature($payload, $signature, $gateway['webhookSecret'])) {
    http_response_code(401);
    exit('Invalid signature');
}

$event = json_decode($payload, true);

switch ($event['type']) {
    case 'charge.confirmed':
        handleCryptoConfirmed($event['data']);
        break;
        
    case 'charge.pending':
        handleCryptoPending($event['data']);
        break;
        
    case 'charge.failed':
        handleCryptoFailed($event['data']);
        break;
        
    case 'charge.expired':
        handleCryptoExpired($event['data']);
        break;
}

http_response_code(200);

function handleCryptoConfirmed($data)
{
    $chargeId = $data['id'];
    $invoiceId = $data['metadata']['invoice_id'];
    $amount = $data['payments'][0]['value']['crypto']['amount'] ?? $data['amount'];
    $currency = $data['currency'] ?? 'USD';
    
    // Calculate using crypto amount and rate at payment time
    $paidAmount = $data['amount_paid'];
    
    addInvoicePayment($invoiceId, $chargeId, $paidAmount, 0, 'crypto');
    
    logTransaction('crypto', $data, 'Payment confirmed');
    
    // Store crypto transaction details
    Capsule::table('mod_crypto_payments')->insert([
        'invoice_id' => $invoiceId,
        'charge_id' => $chargeId,
        'crypto_amount' => $amount,
        'crypto_currency' => $data['currency'],
        'fiat_amount' => $paidAmount,
        'fiat_currency' => $currency,
        'tx_hash' => $data['payments'][0]['tx_hash'] ?? '',
        'status' => 'confirmed',
        'confirmed_at' => date('Y-m-d H:i:s'),
    ]);
}

function handleCryptoPending($data)
{
    $invoiceId = $data['metadata']['invoice_id'];
    logTransaction('crypto', $data, 'Payment pending');
}

function handleCryptoFailed($data)
{
    logTransaction('crypto', $data, 'Payment failed');
}
```

### Step 4: Real-Time Price Conversion
```php
<?php
function getCryptoPrice($crypto, $currency = 'USD')
{
    // Use price API
    $cacheKey = "crypto_price_{$crypto}_{$currency}";
    
    $cached = Capsule::cache()->get($cacheKey);
    if ($cached) {
        return $cached;
    }
    
    $response = file_get_contents("https://api.coingecko.com/api/v3/simple/price?ids=$crypto&vs_currencies=$currency");
    $data = json_decode($response, true);
    
    $price = $data[$crypto][$currency] ?? 0;
    
    Capsule::cache()->put($cacheKey, $price, 300); // Cache 5 minutes
    
    return $price;
}

function calculateCryptoAmount($fiatAmount, $crypto, $currency = 'USD')
{
    $price = getCryptoPrice($crypto, $currency);
    
    return $fiatAmount / $price;
}
```

## Supported Cryptocurrencies
| Currency | Symbol | Network |
|----------|--------|---------|
| Bitcoin | BTC | Bitcoin |
| Ethereum | ETH | Ethereum |
| Litecoin | LTC | Litecoin |
| Bitcoin Cash | BCH | Bitcoin Cash |
| Dogecoin | DOGE | Dogecoin |
| USDC | USDC | Ethereum/Solana |

## Benefits
- No chargebacks
- Fast settlement
- Global reach
- Lower fees (sometimes)
- Privacy

## Tags
- cryptocurrency
- bitcoin
- ethereum
- payment
- crypto