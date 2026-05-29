# WHMCS InvoiceCreated Hook Reference

## Overview

The `InvoiceCreated` hook fires when a new invoice is generated in WHMCS. This hook triggers during invoice creation for orders, renewals, and other billing events.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `invoiceid` | int | The unique invoice ID |
| `userid` | int | The client ID |
| `total` | float | Invoice total amount |
| `status` | string | Invoice status (usually Unpaid) |
| `date` | string | Invoice creation date |
| `duedate` | string | Payment due date |
| `model` | object | The Invoice model instance |
| `lineitems` | array | Invoice line items |

## Example Implementation

```php
<?php
add_hook('InvoiceCreated', 1, function(array $params) {
    // Log invoice creation
    logActivity("Invoice #{$params['invoiceid']} created for client #{$params['userid']}");
    
    // Add custom line item for setup fee if applicable
    $invoice = $params['model'];
    foreach ($invoice->lineItems as $item) {
        if ($item->type === 'Hosting' && hasSetupFee($item->relid)) {
            addInvoiceLineItem($params['invoiceid'], 'Setup Fee', 25.00, 'setup');
        }
    }
    
    return $params;
});
```

## Custom Invoice Adjustments

```php
<?php
<?php
add_hook('InvoiceCreated', 1, function(array $params) {
    $userId = (int)$params['userid'];
    
    // 1. Apply discount based on client group
    $client = getClientsDetails($userId);
    if ($client['groupid'] == 2) { // VIP group
        addInvoiceDiscount($params['invoiceid'], 'VIP Discount', 10, 'percent');
    }
    
    // 2. Add late payment fee for overdue clients
    $overdueCount = getOverdueInvoiceCount($userId);
    if ($overdueCount >= 3) {
        addInvoiceLineItem($params['invoiceid'], 'Late Payment Fee', 15.00, 'fee');
    }
    
    // 3. Apply credit balance
    $creditBalance = getCreditBalance($userId);
    if ($creditBalance > 0) {
        $invoiceTotal = $params['total'];
        $creditToApply = min($creditBalance, $invoiceTotal);
        addInvoiceCredit($params['invoiceid'], $creditToApply);
    }
    
    // 4. Add tax override for specific regions
    $clientCountry = $client['country'];
    if (in_array($clientCountry, ['CA', 'MX'])) {
        addCanadianHST($params['invoiceid'], $clientCountry);
    }
    
    return $params;
});
```

## Notification Workflow

```php
<?php
add_hook('InvoiceCreated', 1, function(array $params) {
    // Get client details
    $client = getClientsDetails($params['userid']);
    
    // 1. Send invoice notification via SMS for high-value invoices
    if ($params['total'] >= 500) {
        sendSMSNotification($client['phonenumber'], 
            "Invoice #{$params['invoiceid']} for {$params['total']} is now available.");
    }
    
    // 2. Post to Slack channel for team visibility
    sendSlackMessage([
        'channel' => '#billing',
        'message' => "New invoice created: #{$params['invoiceid']} for " .
                     "{$client['fullname']} - Total: {$params['total']}"
    ]);
    
    // 3. Add to accounting software
    syncInvoiceToAccounting($params['invoiceid']);
    
    // 4. Create internal notification for high-value invoices
    if ($params['total'] >= 1000) {
        createAdminNotification([
            'type' => 'high_value_invoice',
            'message' => "High value invoice created: {$params['invoiceid']}",
            'related_id' => $params['invoiceid']
        ]);
    }
    
    return $params;
});
```

## Validation and Modification

```php
<?php
add_hook('InvoiceCreated', 1, function(array $params) {
    // Prevent invoice creation for clients with overdue balances
    $userId = (int)$params['userid'];
    $overdueBalance = getOverdueBalance($userId);
    
    if ($overdueBalance > 1000) { // High overdue threshold
        return [
            'success' => false,
            'errorMessage' => 'Account has significant overdue balance. Please resolve before new orders.'
        ];
    }
    
    // Add notes for specific product types
    foreach ($params['lineitems'] ?? [] as $item) {
        if ($item['type'] === 'Hosting' && isPremiumProduct($item['relid'])) {
            addInvoiceNote($params['invoiceid'], 'Premium service - priority support included');
        }
    }
    
    return $params;
});
```

## Use Cases

- **Custom Line Items**: Add fees, discounts, or credits
- **Notifications**: Alert customers or staff
- **Integration**: Sync to external systems
- **Validation**: Prevent creation under certain conditions
- **Accounting**: Export to accounting software

## Notes

- Runs after invoice is created but before it's finalized
- Can return false to prevent invoice creation (with error message)
- Line items can be added or modified before finalization
- Consider using `InvoicePaid` for payment-triggered actions

## Related Hooks

- `InvoicePaid` - When invoice is paid
- `InvoiceCancelled` - When invoice is cancelled
- `CalculateCartTotals` - During cart calculation

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Invoice Management](../whmcs-invoice-setup.md)