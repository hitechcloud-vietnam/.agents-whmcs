# WHMCS ACH Payment Setup Workflow

## Description
Configure ACH (Automated Clearing House) payments for US customers.

## Prerequisites
- US business bank account
- ACH-enabled payment gateway
- Merchant services account

## Steps

### Step 1: Get ACH Credentials
```bash
# From your payment processor or bank:
# - API credentials
# - Merchant ID
# - Public key for encryption
```

### Step 2: Create ACH Gateway
```php
<?php
// modules/gateways/ach/ach.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function ach_MetaData()
{
    return [
        'DisplayName' => 'ACH Payment',
        'APIVersion' => '1.0',
        'supportsRecurring' => true,
    ];
}

function ach_config()
{
    return [
        'FriendlyName' => ['value' => 'ACH Payment'],
        'merchantId' => ['Type' => 'text', 'Label' => 'Merchant ID'],
        'apiKey' => ['Type' => 'password', 'Label' => 'API Key'],
        'publicKey' => ['Type' => 'text', 'Label' => 'Public Key'],
        'environment' => ['Type' => 'dropdown', 'Options' => 'sandbox,live'],
    ];
}

function ach_link($params)
{
    $html = '<form id="ach-form" method="post" action="process.php">';
    $html .= '<input type="hidden" name="invoice_id" value="' . $params['invoiceid'] . '">';
    
    $html .= '<div class="ach-fields">';
    $html .= '<label>Bank Name:</label>';
    $html .= '<input type="text" name="bank_name" required>';
    
    $html .= '<label>Routing Number:</label>';
    $html .= '<input type="text" name="routing_number" required pattern="[0-9]{9}" maxlength="9">';
    
    $html .= '<label>Account Number:</label>';
    $html .= '<input type="text" name="account_number" required maxlength="17">';
    
    $html .= '<label>Account Type:</label>';
    $html .= '<select name="account_type" required>';
    $html .= '<option value="checking">Checking</option>';
    $html .= '<option value="savings">Savings</option>';
    $html .= '</select>';
    
    $html .= '<label>Account Holder Name:</label>';
    $html .= '<input type="text" name="account_holder" required>';
    $html .= '</div>';
    
    $html .= '<div class="ach-authorization">';
    $html .= '<input type="checkbox" name="ach_agreement" required>';
    $html .= '<span>I authorize this payment from my bank account.</span>';
    $html .= '</div>';
    
    $html .= '<button type="submit" class="btn btn-primary">Pay with Bank</button>';
    $html .= '</form>';
    
    return $html;
}
```

### Step 3: Process ACH Payment
```php
<?php
// modules/gateways/callback/ach/process.php

function processAchPayment($params)
{
    $gateway = getGatewayVariables('ach');
    
    // Validate routing number
    if (!validateACH RoutingNumber($_POST['routing_number'])) {
        return ['error' => 'Invalid routing number'];
    }
    
    // Encrypt account details
    $encryptedAccount = encryptBankAccount($_POST['account_number'], $gateway['publicKey']);
    
    // Create ACH transaction
    $result = createACHTransaction([
        'merchant_id' => $gateway['merchantId'],
        'routing_number' => $_POST['routing_number'],
        'account_number' => $_POST['account_number'],
        'account_type' => $_POST['account_type'],
        'account_holder' => $_POST['account_holder'],
        'amount' => $params['amount'],
        'invoice_id' => $params['invoiceid'],
    ]);
    
    return $result;
}

function validateACH RoutingNumber($routing)
{
    // ACH routing number validation using checksum
    if (strlen($routing) !== 9 || !ctype_digit($routing)) {
        return false;
    }
    
    // Federal Reserve routing symbol
    $symbol = substr($routing, 0, 4);
    $validPrefixes = range(0, 12); // Valid first digit combinations
    
    return true; // Basic validation - implement full checksum
}
```

### Step 4: ACH Settlement
```php
<?php
// ACH typically takes 2-5 business days to settle

function handleACHSettlement($transactionId)
{
    $transaction = Capsule::table('tblaccounts')
        ->where('transid', $transactionId)
        ->first();
    
    if ($transaction && $transaction->amount > 0) {
        // Mark payment as settled (ACH is already recorded, just update status)
        Capsule::table('mod_ach_transactions')
            ->where('transaction_id', $transactionId)
            ->update(['status' => 'settled', 'settled_at' => date('Y-m-d H:i:s')]);
        
        logActivity("ACH payment settled: $transactionId");
    }
}
```

### Step 5: Handle ACH Returns
```php
<?php
// Handle ACH return notifications

function handleACHReturn($returnData)
{
    $transactionId = $returnData['original_transaction_id'];
    $returnCode = $returnData['return_code'];
    $returnReason = $returnData['return_reason'];
    
    // Find the original transaction
    $transaction = Capsule::table('tblaccounts')
        ->where('transid', $transactionId)
        ->first();
    
    if ($transaction) {
        // Reverse the payment
        Capsule::table('tblaccounts')->insert([
            'userid' => $transaction->userid,
            'currency' => $transaction->currency,
            'amount' => -$transaction->amount,
            'description' => "ACH Return: $returnReason",
            'transid' => $transactionId . '_RETURN',
            'invoiceid' => $transaction->invoiceid,
            'date' => date('Y-m-d H:i:s'),
        ]);
        
        // Update invoice status
        Capsule::table('tblinvoices')
            ->where('id', $transaction->invoiceid)
            ->update(['status' => 'Unpaid']);
        
        // Suspend service if applicable
        suspendServiceByInvoice($transaction->invoiceid);
        
        // Log and notify
        logActivity("ACH return processed for transaction $transactionId: $returnReason");
        
        sendEmail('ACHPaymentReturned', $transaction->userid, [
            'invoice_id' => $transaction->invoiceid,
            'amount' => $transaction->amount,
            'reason' => $returnReason,
        ]);
    }
}
```

## ACH Return Codes
| Code | Reason |
|------|--------|
| R01 | Insufficient funds |
| R02 | Account closed |
| R04 | Invalid account number |
| R08 | Payment stopped |
| R10 | Customer advises not authorized |
| R14 | Representative payee deceased |
| R29 | Corporate account holder deceased |

## Tags
- ach
- bank-payment
- us-payments
- payment