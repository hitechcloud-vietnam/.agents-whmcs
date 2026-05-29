# WHMCS InvoiceCancelled Hook Reference

## Overview

The `InvoiceCancelled` hook fires when an invoice is cancelled in WHMCS. This hook triggers when an admin or automated process cancels an invoice, typically releasing any reserved inventory or resources.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `invoiceid` | int | The unique invoice ID |
| `userid` | int | The client ID |
| `total` | float | Invoice total amount |
| `status` | string | Invoice status (Cancelled) |
| `date` | string | Original invoice date |
| `model` | object | The Invoice model instance |

## Example Implementation

```php
<?php
add_hook('InvoiceCancelled', 1, function(array $params) {
    // Log the cancellation
    logActivity("Invoice #{$params['invoiceid']} cancelled for client #{$params['userid']}");
    
    // Notify the customer
    $client = getClientsDetails($params['userid']);
    sendTemplatedEmail('Invoice Cancelled', $client['email'], [
        'invoice_id' => $params['invoiceid'],
        'reason' => 'Cancelled by administrator'
    ]);
    
    return $params;
});
```

## Inventory Release

```php
<?php
add_hook('InvoiceCancelled', 1, function(array $params) {
    // Get line items to determine what to release
    $result = full_query("
        SELECT * FROM tblinvoiceitems 
        WHERE invoiceid = " . (int)$params['invoiceid']
    );
    
    while ($item = mysql_fetch_array($result)) {
        switch ($item['type']) {
            case 'Hosting':
                // Don't provision - service remains on hold
                logActivity("Service {$item['relid']} not provisioned due to cancelled invoice");
                break;
                
            case 'Domain':
                // Mark domain as not paid, don't register
                update_query('tbldomains', [
                    'status' => 'Pending',
                    'registrar' => ''
                ], ['id' => $item['relid']]);
                break;
                
            case 'Addon':
                // Don't activate addon
                break;
        }
    }
    
    // Release any reserved inventory
    releaseReservedInventory($params['invoiceid']);
    
    return $params;
});
```

## Refund Processing

```php
<?php
add_hook('InvoiceCancelled', 1, function(array $params) {
    $invoiceId = (int)$params['invoiceid'];
    
    // Check if payment was already made
    $payments = getInvoicePayments($invoiceId);
    
    if (!empty($payments)) {
        foreach ($payments as $payment) {
            // Initiate refund if paid via certain gateways
            if (in_array($payment['gateway'], ['stripe', 'paypal'])) {
                processRefund($payment['transactionid'], $payment['amount']);
                
                // Log refund
                logActivity("Refund initiated for transaction {$payment['transactionid']}");
            }
        }
        
        // Create refund record
        insert_query('tbl_refund_history', [
            'invoice_id' => $invoiceId,
            'amount' => $params['total'],
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    return $params;
});
```

## Service Status Management

```php
<?php
add_hook('InvoiceCancelled', 1, function(array $params) {
    $invoiceId = (int)$params['invoiceid'];
    
    // Find associated services that might be pending activation
    $services = full_query("
        SELECT h.id, h.domain, h.status 
        FROM tblhosting h
        INNER JOIN tblinvoiceitems ii ON ii.relid = h.id AND ii.type = 'Hosting'
        WHERE ii.invoiceid = $invoiceId
    ");
    
    while ($service = mysql_fetch_array($services)) {
        // Keep service on pending if it hasn't been activated yet
        if ($service['status'] === 'Pending') {
            // Optionally terminate the pending service entirely
            // update_query('tblhosting', ['status' => 'Cancelled'], ['id' => $service['id']]);
            
            logActivity("Service {$service['id']} ({$service['domain']}) awaiting activation cancelled");
        }
    }
    
    // Notify fulfillment team
    sendAdminNotification('system', [
        'subject' => "Invoice Cancelled - Fulfillment Check Required",
        'message' => "Invoice #{$invoiceId} has been cancelled. " .
                     "Please verify no services were provisioned."
    ]);
    
    return $params;
});
```

## Use Cases

- **Inventory Management**: Release reserved resources
- **Notification**: Alert customers and staff
- **Refund Processing**: Handle refunds for paid invoices
- **Service Management**: Update service statuses
- **Audit Trail**: Maintain records of cancellations

## Notes

- Runs after invoice status changes to Cancelled
- May need to handle both paid and unpaid invoice cancellations
- Consider refund workflows for paid invoices
- Services may need status updates based on cancellation

## Related Hooks

- `InvoiceCreated` - When invoice is created
- `InvoicePaid` - When invoice is paid
- `ServiceDelete` - When services are deleted

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Invoice Management](../whmcs-invoice-setup.md)