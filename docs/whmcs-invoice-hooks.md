# WHMCS Invoice Hooks

## Overview

Invoice hooks allow you to intercept and modify invoice-related operations in WHMCS.

## Available Invoice Hooks

### Invoice Created Hook

```php
<?php
// Triggered when an invoice is created
add_hook('InvoiceCreated', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];
    $userId = $vars['user_id'];
    $total = $vars['total'];
    
    // Apply automatic discounts
    applyInvoiceDiscounts($invoiceId);
    
    // Add late fee if applicable
    checkAndApplyLateFees($invoiceId, $userId);
    
    // Sync to accounting software
    syncInvoiceToAccounting($invoiceId);
    
    // Set custom due date
    setCustomDueDate($invoiceId);
    
    return ['success' => true, 'invoice_id' => $invoiceId];
});

function syncInvoiceToAccounting(int $invoiceId): void
{
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->first();
    
    $items = Capsule::table('tblinvoiceitems')
        ->where('invoice_id', $invoiceId)
        ->get();
    
    // Send to external accounting system
    $accountingService->createInvoice([
        'external_id' => 'WHMCS-' . $invoiceId,
        'client_id' => $invoice->userid,
        'amount' => $invoice->total,
        'due_date' => $invoice->duedate,
        'items' => $items,
    ]);
}
```

### Invoice Paid Hook

```php
<?php
// Triggered when an invoice is paid
add_hook('InvoicePaid', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];
    $paymentAmount = $vars['amount_paid'];
    $paymentMethod = $vars['payment_method'];
    
    // Update accounting
    recordPaymentInAccounting($invoiceId, $vars['transid']);
    
    // Unlock suspended services
    unlockSuspendedServices($invoiceId);
    
    // Send confirmation
    sendPaymentConfirmation($invoiceId);
    
    // Trigger loyalty points
    awardLoyaltyPoints($vars['user_id'], $paymentAmount);
    
    // Process affiliate commissions
    processAffiliateCommission($invoiceId);
    
    return ['success' => true];
});

function unlockSuspendedServices(int $invoiceId): void
{
    $services = Capsule::table('tblhosting')
        ->where('invoice_id', $invoiceId)
        ->where('domainstatus', 'Suspended')
        ->get();
    
    foreach ($services as $service) {
        // Call module unsuspend
        $module = new Module();
        $module->load($service->servertype);
        $module->unsuspendService($service->id);
        
        // Update status
        Capsule::table('tblhosting')
            ->where('id', $service->id)
            ->update(['domainstatus' => 'Active']);
    }
}
```

### Invoice Overdue Hook

```php
<?php
// Triggered when an invoice becomes overdue
add_hook('InvoiceOverdueReminder', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];
    $daysOverdue = $vars['days_overdue'];
    $userId = $vars['user_id'];
    
    // Send escalating reminders
    sendEscalatingReminders($invoiceId, $daysOverdue);
    
    // Apply overdue penalty
    if ($daysOverdue >= 7) {
        applyOverduePenalty($invoiceId);
    }
    
    // Suspend services if past threshold
    if ($daysOverdue >= getConfig('auto_suspend_days')) {
        suspendOverdueServices($userId);
    }
    
    // Notify sales team
    notifySalesTeam($invoiceId, $daysOverdue);
    
    return ['success' => true];
});

function sendEscalatingReminders(int $invoiceId, int $daysOverdue): void
{
    $templates = [
        1 => 'Overdue Notice - Day 1',
        7 => 'Overdue Notice - Day 7',
        14 => 'Final Warning - Day 14',
        30 => 'Collection Notice - Day 30',
    ];
    
    if (isset($templates[$daysOverdue])) {
        sendTemplatedEmail($userId, $templates[$daysOverdue], [
            'invoice_id' => $invoiceId,
            'days_overdue' => $daysOverdue,
        ]);
    }
}
```

### Invoice Cancelled Hook

```php
<?php
// Triggered when an invoice is cancelled
add_hook('InvoiceCancelled', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];
    $reason = $vars['reason'] ?? 'No reason provided';
    
    // Reverse accounting entry
    reverseAccountingEntry($invoiceId);
    
    // Restore suspended services if needed
    restoreServicesIfApplicable($invoiceId);
    
    // Send cancellation notice
    sendInvoiceCancellationNotice($invoiceId, $reason);
    
    // Log for audit
    logInvoiceCancellation($invoiceId, $reason);
    
    return ['success' => true];
});
```

### Invoice Refunded Hook

```php
<?php
// Triggered when an invoice is refunded
add_hook('InvoiceRefunded', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];
    $refundAmount = $vars['refund_amount'];
    $refundMethod = $vars['refund_method'];
    $userId = $vars['user_id'];
    
    // Record refund in accounting
    recordRefundInAccounting($invoiceId, $refundAmount, $refundMethod);
    
    // Reverse loyalty points
    reverseLoyaltyPoints($userId, $refundAmount);
    
    // Reverse affiliate commission
    reverseAffiliateCommission($invoiceId);
    
    // Send refund confirmation
    sendRefundConfirmation($invoiceId, $refundAmount);
    
    return ['success' => true];
});
```

### Pre-Invoice Creation Hook

```php
<?php
// Modify invoice before creation
add_hook('PreInvoiceCreate', 1, function(array $vars) {
    $userId = $vars['user_id'];
    $items = $vars['items'];
    
    // Add handling fee for certain payment methods
    if ($vars['payment_method'] === 'banktransfer') {
        $items[] = [
            'description' => 'Bank Transfer Processing Fee',
            'amount' => 2.50,
            'taxed' => true,
        ];
    }
    
    // Apply volume discount
    $discount = calculateVolumeDiscount($userId);
    if ($discount > 0) {
        $items[] = [
            'description' => 'Volume Discount',
            'amount' => -$discount,
            'taxed' => false,
        ];
    }
    
    return [
        'abort' => false,
        'items' => $items,
        'notes' => 'Custom invoice notes',
    ];
});
```

### Invoice Display Hook

```php
<?php
// Modify invoice display
add_hook('InvoiceDisplayPreExit', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];
    
    // Add custom fields to invoice
    $customData = getCustomInvoiceData($invoiceId);
    
    // Add payment instructions
    $bankDetails = getBankTransferDetails($vars['user_id']);
    
    return [
        'custom_data' => $customData,
        'bank_details' => $bankDetails,
        'footer_message' => 'Thank you for your business!',
    ];
});
```

## Invoice Item Hooks

```php
<?php
// Triggered when invoice items are added
add_hook('InvoiceAddItem', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];
    $itemType = $vars['type'];
    $itemId = $vars['rel_id'];
    
    // Track item for analytics
    trackInvoiceItem($invoiceId, $itemType, $itemId);
    
    // Add item-specific notes
    addItemNotes($invoiceId, $itemType, $itemId);
    
    return ['success' => true];
});

// Triggered when invoice item is deleted
add_hook('InvoiceRemoveItem', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];
    $itemId = $vars['item_id'];
    
    // Log item removal
    logItemRemoval($invoiceId, $itemId);
    
    return ['success' => true];
});
```

## Comprehensive Invoice Handler

```php
<?php
class InvoiceHookHandler {
    
    public function register(): void
    {
        add_hook('InvoiceCreated', 1, [$this, 'handleInvoiceCreated']);
        add_hook('InvoicePaid', 1, [$this, 'handleInvoicePaid']);
        add_hook('InvoiceOverdueReminder', 1, [$this, 'handleInvoiceOverdue']);
        add_hook('InvoiceCancelled', 1, [$this, 'handleInvoiceCancelled']);
        add_hook('InvoiceRefunded', 1, [$this, 'handleInvoiceRefunded']);
        add_hook('PreInvoiceCreate', 1, [$this, 'handlePreInvoiceCreate']);
    }
    
    public function handleInvoiceCreated(array $vars): array
    {
        $invoiceId = $vars['invoice_id'];
        
        // Set payment terms based on client
        $this->setPaymentTerms($invoiceId, $vars['user_id']);
        
        // Add default items
        $this->addDefaultItems($invoiceId);
        
        // Send initial invoice
        $this->queueInitialEmail($invoiceId);
        
        return ['success' => true];
    }
    
    public function handleInvoicePaid(array $vars): array
    {
        $invoiceId = $vars['invoice_id'];
        
        // Unlock services
        $this->unlockServices($invoiceId);
        
        // Record payment
        $this->recordPayment($invoiceId, $vars);
        
        // Award points/credits
        $this->awardIncentives($vars['user_id'], $vars['amount_paid']);
        
        // Notify team
        $this->notifyTeam($invoiceId);
        
        return ['success' => true];
    }
    
    public function handleInvoiceOverdue(array $vars): array
    {
        $daysOverdue = $vars['days_overdue'];
        
        if ($daysOverdue >= 3) {
            $this->escalateToSales($vars['invoice_id']);
        }
        
        if ($daysOverdue >= 7) {
            $this->applyLateFee($vars['invoice_id']);
        }
        
        if ($daysOverdue >= 14) {
            $this->suspendServices($vars['user_id']);
        }
        
        return ['success' => true];
    }
    
    public function handleInvoiceCancelled(array $vars): array
    {
        $this->logCancellation($vars['invoice_id'], $vars['reason'] ?? '');
        return ['success' => true];
    }
    
    public function handleInvoiceRefunded(array $vars): array
    {
        $this->reversePayment($vars['invoice_id'], $vars['refund_amount']);
        $this->notifyRefunded($vars['invoice_id'], $vars['refund_amount']);
        return ['success' => true];
    }
    
    public function handlePreInvoiceCreate(array $vars): array
    {
        // Custom invoice preparation
        return [
            'abort' => false,
            'items' => $vars['items'],
        ];
    }
    
    private function setPaymentTerms(int $invoiceId, int $userId): void
    {
        $client = Capsule::table('tblclients')->where('id', $userId)->first();
        
        if ($client->credit > 100) {
            // VIP terms: net 30
            Capsule::table('tblinvoices')
                ->where('id', $invoiceId)
                ->update(['duedate' => date('Y-m-d', strtotime('+30 days'))]);
        }
    }
    
    private function addDefaultItems(int $invoiceId): void
    {
        // Add insurance or warranty items
    }
    
    private function queueInitialEmail(int $invoiceId): void
    {
        Capsule::table('mod_email_queue')->insert([
            'invoice_id' => $invoiceId,
            'template' => 'invoice_created',
            'queued_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function unlockServices(int $invoiceId): void
    {
        $services = Capsule::table('tblhosting')
            ->where('invoice_id', $invoiceId)
            ->where('domainstatus', 'Suspended')
            ->get();
        
        foreach ($services as $service) {
            // Unsuspend logic
        }
    }
    
    private function recordPayment(int $invoiceId, array $vars): void
    {
        Capsule::table('mod_payment_records')->insert([
            'invoice_id' => $invoiceId,
            'amount' => $vars['amount_paid'],
            'method' => $vars['payment_method'],
            'transaction_id' => $vars['transid'] ?? '',
            'recorded_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function awardIncentives(int $userId, float $amount): void
    {
        $points = floor($amount);
        Capsule::table('mod_loyalty_points')
            ->where('client_id', $userId)
            ->increment('points', $points);
    }
    
    private function notifyTeam(int $invoiceId): void
    {
        // Notify sales team of payment
    }
    
    private function escalateToSales(int $invoiceId): void
    {
        // Create task for sales
    }
    
    private function applyLateFee(int $invoiceId): void
    {
        // Add late fee to invoice
    }
    
    private function suspendServices(int $userId): void
    {
        // Suspend overdue services
    }
    
    private function logCancellation(int $invoiceId, string $reason): void
    {
        Capsule::table('mod_invoice_audit')->insert([
            'invoice_id' => $invoiceId,
            'action' => 'cancelled',
            'reason' => $reason,
            'logged_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function reversePayment(int $invoiceId, float $amount): void
    {
        // Reverse accounting entry
    }
    
    private function notifyRefunded(int $invoiceId, float $amount): void
    {
        // Send refund notification
    }
}

$handler = new InvoiceHookHandler();
$handler->register();
```

## Best Practices

1. **Avoid infinite loops** - Don't modify invoices in their own hooks
2. **Handle edge cases** - Check for deleted/missing data
3. **Use transactions** - Wrap multiple operations
4. **Log everything** - Maintain audit trail
5. **Queue background work** - Don't slow down requests

## Related Documentation

- [WHMCS Order Hooks](/docs/whmcs-order-hooks.md)
- [WHMCS Payment Hooks](/docs/whmcs-payment-hooks.md)