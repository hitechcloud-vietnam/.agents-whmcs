# WHMCS Local Payment Gateway Workflow

## Description
Create local/offline payment gateway for bank transfers and cash payments.

## Steps

### Step 1: Create Local Gateway
```php
<?php
// modules/gateways/bank_transfer/bank_transfer.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function bank_transfer_MetaData()
{
    return [
        'DisplayName' => 'Bank Transfer',
        'APIVersion' => '1.0',
    ];
}

function bank_transfer_config()
{
    return [
        'FriendlyName' => ['value' => 'Bank Transfer'],
        'bankName' => ['Type' => 'text', 'Label' => 'Bank Name'],
        'accountNumber' => ['Type' => 'text', 'Label' => 'Account Number'],
        'accountName' => ['Type' => 'text', 'Label' => 'Account Name'],
        'branch' => ['Type' => 'text', 'Label' => 'Branch'],
        'instructions' => [
            'Type' => 'textarea',
            'Label' => 'Payment Instructions',
            'Description' => 'Instructions shown to customer'
        ],
        'autoComplete' => ['Type' => 'yesno', 'Label' => 'Auto-complete after X days'],
    ];
}

function bank_transfer_link($params)
{
    $instructions = nl2br($params['instructions']);
    
    $html = '<div class="bank-transfer-info">';
    $html .= '<h4>Bank Transfer Details</h4>';
    $html .= '<p><strong>Bank:</strong> ' . htmlspecialchars($params['bankName']) . '</p>';
    $html .= '<p><strong>Account Number:</strong> ' . htmlspecialchars($params['accountNumber']) . '</p>';
    $html .= '<p><strong>Account Name:</strong> ' . htmlspecialchars($params['accountName']) . '</p>';
    $html .= '<p><strong>Branch:</strong> ' . htmlspecialchars($params['branch']) . '</p>';
    $html .= '<p><strong>Amount:</strong> ' . formatCurrency($params['amount']) . '</p>';
    $html .= '<p><strong>Reference:</strong> INV' . $params['invoiceid'] . '</p>';
    $html .= '<hr>';
    $html .= '<p>' . $instructions . '</p>';
    $html .= '</div>';
    
    return $html;
}
```

### Step 2: Create Manual Verification
```php
<?php
// Create admin interface for manual verification
// Add to addon module or admin area

function verify_manual_payment($invoiceId, $amount, $reference, $clientId)
{
    // Add payment
    $transactionId = 'BT_' . date('Ymd') . '_' . rand(1000, 9999);
    
    addInvoicePayment($invoiceId, $transactionId, $amount, 0, 'bank_transfer');
    
    logActivity("Manual bank transfer verified for invoice #$invoiceId");
    
    return ['success' => true, 'transaction_id' => $transactionId];
}
```

## Bank Transfer Workflow
1. Client selects bank transfer
2. Client sees payment details
3. Client transfers money to your account
4. You manually verify payment in admin
5. WHMCS marks invoice as paid

## Tags
- local-payment
- bank-transfer
- manual-payment
- offline