# WHMCS Credit Management

## Concept Explanation

Credit management in WHMCS handles client account credits, store credit balances, and credit applications. Credits can be manually added, automatically applied to invoices, or accumulated through overpayments, refunds, or promotional offers. Effective credit management improves cash flow and customer satisfaction.

### Credit Types

- **Manual Credits**: Added by admin for adjustments
- **Overpayment Credits**: Excess payments stored as credit
- **Refund Credits**: Refunds converted to account balance
- **Promotional Credits**: Marketing/promotional credit allocations
- **Loyalty Credits**: Points/rewards converted to credit

## Code Patterns & Templates

### Credit Manager Class

```php
<?php
// includes/CreditManager.php

namespace WHMCS\Credit;

class CreditManager {
    
    /**
     * Add credit to client account
     */
    public static function addCredit($clientId, $amount, $description, $type = 'manual', $invoiceId = 0) {
        update_query('tblclients', [
            'credit' => "credit + " . (float)$amount
        ], ['id' => $clientId]);
        
        insert_query('tblcredit', [
            'clientid' => $clientId,
            'date' => date('Y-m-d H:i:s'),
            'description' => $description,
            'amount' => $amount,
            'type' => $type,
            'invoice_id' => $invoiceId
        ]);
        
        sendEmailTemplate('credit_added', $clientId, [
            'amount' => formatCurrency($amount),
            'balance' => self::getCreditBalance($clientId),
            'description' => $description
        ]);
        
        logActivity("Credit of {$amount} added for client {$clientId}: {$description}", $clientId);
        return true;
    }
    
    public static function getCreditBalance($clientId) {
        $result = full_query("SELECT credit FROM tblclients WHERE id = " . (int)$clientId);
        $row = mysql_fetch_array($result);
        return (float)($row['credit'] ?? 0);
    }
    
    public static function applyToInvoice($invoiceId, $clientId, $amount = null) {
        $invoice = getInvoice($invoiceId);
        $balance = self::getCreditBalance($clientId);
        $amount = $amount ?? min($balance, $invoice['total']);
        
        if ($amount > $invoice['total']) $amount = $invoice['total'];
        
        update_query('tblclients', ['credit' => "credit - " . (float)$amount], ['id' => $clientId]);
        addInvoicePayment($invoiceId, 'credit_' . time(), $amount, 0, 'credit');
        
        if (getInvoiceBalance($invoiceId) <= 0) {
            updateInvoiceStatus($invoiceId, 'Paid');
        }
        
        return $amount;
    }
    
    public static function transferCredit($fromClientId, $toClientId, $amount, $reason = '') {
        if ($fromClientId === $toClientId) {
            throw new \Exception("Cannot transfer credit to same account");
        }
        self::addCredit($toClientId, $amount, "Credit transfer from Client #{$fromClientId}: {$reason}");
        self::deductCredit($fromClientId, $amount, "Credit transfer to Client #{$toClientId}: {$reason}");
        return true;
    }
    
    public static function deductCredit($clientId, $amount, $description) {
        $currentBalance = self::getCreditBalance($clientId);
        if ($currentBalance < $amount) {
            throw new \Exception("Insufficient credit balance");
        }
        update_query('tblclients', ['credit' => "credit - " . (float)$amount], ['id' => $clientId]);
        insert_query('tblcredit', [
            'clientid' => $clientId, 'date' => date('Y-m-d H:i:s'),
            'description' => $description, 'amount' => -$amount, 'type' => 'debit'
        ]);
        return true;
    }
    
    public static function getTransactionHistory($clientId, $limit = 50) {
        $result = full_query("SELECT * FROM tblcredit WHERE clientid = " . (int)$clientId . " ORDER BY date DESC LIMIT " . (int)$limit);
        $transactions = [];
        while ($row = mysql_fetch_array($result)) { $transactions[] = $row; }
        return $transactions;
    }
}
```

### Credit Auto-Application Hook

```php
<?php
// hooks/credit_auto_apply.php

add_hook('InvoiceCreationPreEmail', 1, function($vars) {
    $invoice = getInvoice($vars['invoiceid']);
    if (strpos($invoice['notes'], 'noautocredit') !== false) return $vars;
    
    $clientId = $invoice['userid'];
    $creditBalance = \WHMCS\Credit\CreditManager::getCreditBalance($clientId);
    
    if ($creditBalance > 0 && $invoice['total'] > 0 && $invoice['total'] <= 500) {
        $applyAmount = min($creditBalance, $invoice['total']);
        try {
            $applied = \WHMCS\Credit\CreditManager::applyToInvoice($invoice['id'], $clientId);
            logActivity("Auto-applied {$applied} credit to invoice #{$invoice['id']}", $clientId);
        } catch (\Exception $e) {
            logActivity("Credit auto-apply failed: " . $e->getMessage(), $clientId);
        }
    }
    return $vars;
});

add_hook('InvoicePaymentReceived', 1, function($vars) {
    $invoice = getInvoice($vars['invoiceid']);
    $overpayment = $vars['amount'] - $invoice['total'];
    
    if ($overpayment > 0.01) {
        \WHMCS\Credit\CreditManager::addCredit(
            $invoice['userid'], $overpayment,
            "Overpayment on Invoice #{$invoice['invoicenum']} - stored as account credit",
            'overpayment', $vars['invoiceid']
        );
    }
});
```

## Step-by-Step Implementation

1. Create database table `tblcredit`
2. Copy CreditManager.php to /includes/
3. Install hooks in /hooks/credit_auto_apply.php
4. Configure auto-application thresholds
5. Test credit operations

## Examples

### Bulk Credit Import

```php
function bulkImportCredits($csvFile) {
    $handle = fopen($csvFile, 'r');
    $imported = $failed = 0;
    while (($data = fgetcsv($handle)) !== false) {
        list($clientId, $amount, $description) = $data;
        try {
            \WHMCS\Credit\CreditManager::addCredit((int)$clientId, (float)$amount, $description);
            $imported++;
        } catch (\Exception $e) { $failed++; }
    }
    fclose($handle);
    return ['imported' => $imported, 'failed' => $failed];
}
```

## Implementation Checklist

- [ ] Create credit transaction table
- [ ] Install CreditManager class
- [ ] Configure auto-application settings
- [ ] Set up credit expiry policy
- [ ] Create admin interface module
- [ ] Configure notification templates
- [ ] Test all credit operations
