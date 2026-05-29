# WHMCS SEPA Direct Debit Setup Workflow

## Description
Configure SEPA Direct Debit payments for European customers.

## Prerequisites
- SEPA-enabled payment gateway
- Business account with SEPA capability
- IBAN collection

## Steps

### Step 1: Get SEPA Credentials
```bash
# Obtain from your bank or payment service provider:
# - SEPA Creditor ID
# - Merchant Account (IBAN)
# - API credentials for gateway
```

### Step 2: Create SEPA Gateway
```php
<?php
// modules/gateways/sepa_direct_debit/sepa.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function sepa_direct_debit_MetaData()
{
    return [
        'DisplayName' => 'SEPA Direct Debit',
        'APIVersion' => '1.0',
        'supportsRecurring' => true,
    ];
}

function sepa_direct_debit_config()
{
    return [
        'FriendlyName' => ['value' => 'SEPA Direct Debit'],
        'creditorId' => ['Type' => 'text', 'Label' => 'Creditor ID'],
        'creditorName' => ['Type' => 'text', 'Label' => 'Creditor Name'],
        'iban' => ['Type' => 'text', 'Label' => 'Creditor IBAN'],
        'apiKey' => ['Type' => 'password', 'Label' => 'API Key'],
        'environment' => ['Type' => 'dropdown', 'Options' => 'sandbox,live'],
    ];
}

function sepa_direct_debit_link($params)
{
    $html = '<form id="sepa-form" method="post" action="process.php">';
    $html .= '<input type="hidden" name="invoice_id" value="' . $params['invoiceid'] . '">';
    
    $html .= '<div class="sepa-fields">';
    $html .= '<label>IBAN:</label>';
    $html .= '<input type="text" name="iban" required pattern="[A-Z]{2}[0-9]{2}[A-Z0-9]+" maxlength="34">';
    $html .= '<label>Account Holder Name:</label>';
    $html .= '<input type="text" name="account_name" required>';
    $html .= '</div>';
    
    $html .= '<div class="sepa-mandate">';
    $html .= '<input type="checkbox" name="mandate_accept" required>';
    $html .= '<span>I authorize ' . htmlspecialchars($params['creditorName']) . ' to collect payments from my account.</span>';
    $html .= '</div>';
    
    $html .= '<button type="submit" class="btn btn-primary">Pay with SEPA</button>';
    $html .= '</form>';
    
    return $html;
}
```

### Step 3: Process SEPA Payment
```php
<?php
// modules/gateways/callback/sepa_direct_debit/process.php

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../../../includes/gatewayfunctions.php';

$gateway = getGatewayVariables('sepa_direct_debit');

$iban = $_POST['iban'];
$accountName = $_POST['account_name'];
$invoiceId = $_POST['invoice_id'];

// Validate IBAN
if (!sepaValidateIBAN($iban)) {
    header('Location: viewinvoice.php?id=' . $invoiceId . '&error=invalid_iban');
    exit;
}

// Get invoice details
$invoice = Capsule::table('tblinvoices')->find($invoiceId);
$client = Capsule::table('tblclients')->find($invoice->userid);

// Create SEPA mandate
$result = createSepaMandate($gateway, [
    'iban' => $iban,
    'account_name' => $accountName,
    'email' => $client->email,
    'creditor_id' => $gateway['creditorId'],
]);

if ($result['success']) {
    // Create payment
    $paymentResult = createSepaPayment($gateway, [
        'mandate_id' => $result['mandate_id'],
        'amount' => $invoice->total,
        'currency' => 'EUR',
        'reference' => 'INV' . $invoiceId,
    ]);
    
    if ($paymentResult['accepted']) {
        // Schedule direct debit
        addInvoicePayment($invoiceId, $paymentResult['transaction_id'], $invoice->total, 0, 'sepa_direct_debit');
        header('Location: viewinvoice.php?id=' . $invoiceId . '&success=1');
    } else {
        header('Location: viewinvoice.php?id=' . $invoiceId . '&error=' . $paymentResult['reason']);
    }
} else {
    header('Location: viewinvoice.php?id=' . $invoiceId . '&error=' . $result['error']);
}
```

### Step 4: IBAN Validation
```php
<?php
function sepaValidateIBAN($iban)
{
    // Remove spaces and convert to uppercase
    $iban = strtoupper(str_replace(' ', '', $iban));
    
    // Check length (varies by country)
    $lengths = [
        'AL' => 28, 'AD' => 24, 'AT' => 20, 'AZ' => 28, 'BH' => 22,
        'BE' => 16, 'BA' => 20, 'BR' => 29, 'BG' => 22, 'CR' => 22,
        'HR' => 21, 'CY' => 28, 'CZ' => 24, 'DK' => 18, 'DO' => 28,
        'TL' => 23, 'EE' => 20, 'FO' => 18, 'FI' => 18, 'FR' => 27,
        'GE' => 22, 'DE' => 22, 'GI' => 23, 'GR' => 27, 'GL' => 18,
        'GT' => 28, 'HU' => 28, 'IS' => 26, 'IQ' => 23, 'IE' => 22,
        'IL' => 23, 'IT' => 27, 'JO' => 30, 'KZ' => 20, 'XK' => 20,
        'KW' => 30, 'LV' => 21, 'LB' => 28, 'LI' => 21, 'LT' => 20,
        'LU' => 20, 'MK' => 19, 'MT' => 31, 'MR' => 27, 'MU' => 30,
        'MC' => 27, 'MD' => 24, 'ME' => 22, 'NL' => 18, 'NO' => 15,
        'PK' => 24, 'PS' => 29, 'PL' => 28, 'PT' => 25, 'QA' => 29,
        'RO' => 24, 'LC' => 32, 'SM' => 27, 'ST' => 25, 'SA' => 24,
        'RS' => 22, 'SC' => 31, 'SK' => 24, 'SI' => 19, 'ES' => 24,
        'SE' => 24, 'CH' => 21, 'TN' => 24, 'TR' => 26, 'UA' => 29,
        'AE' => 23, 'GB' => 22, 'VA' => 22, 'VG' => 24,
    ];
    
    $country = substr($iban, 0, 2);
    
    if (!isset($lengths[$country]) || strlen($iban) !== $lengths[$country]) {
        return false;
    }
    
    // Rearrange and validate checksum
    $rearranged = substr($iban, 4) . substr($iban, 0, 4);
    $numeric = '';
    
    for ($i = 0; $i < strlen($rearranged); $i++) {
        $char = $rearranged[$i];
        if (is_numeric($char)) {
            $numeric .= $char;
        } else {
            $numeric .= (ord($char) - 55); // A=10, B=11, etc.
        }
    }
    
    return bcmod($numeric, 97) === '1';
}
```

### Step 5: Handle SEPA Webhooks
```php
<?php
// modules/gateways/callback/sepa_direct_debit/webhook.php

// Handle SEPA status notifications
function handleSepaWebhook($data)
{
    switch ($data['event']) {
        case 'mandate.created':
            storeMandate($data['mandate_id'], $data);
            break;
            
        case 'payment.settled':
            // Payment has been collected
            $transactionId = $data['payment_id'];
            $amount = $data['amount'];
            logTransaction('sepa_direct_debit', $data, 'Settlement received');
            break;
            
        case 'payment.failed':
            // Payment collection failed
            $reason = $data['fail_reason'];
            logTransaction('sepa_direct_debit', $data, 'Payment failed: ' . $reason);
            // Handle failure
            break;
            
        case 'refund':
            // Handle SEPA refund
            processSepaRefund($data);
            break;
    }
}
```

## SEPA Requirements
1. Minimum 14 days notice before first debit (first payment)
2. Creditor identifier required
3. Mandate required for each debtor
4. Clear refund rights for customers
5. ISO 20022 format for messages

## Supported Countries
SEPA covers EU member states plus: Andorra, Iceland, Liechtenstein, Monaco, Norway, San Marino, Switzerland, United Kingdom, Vatican City

## Tags
- sepa
- direct-debit
- european-payments
- payment