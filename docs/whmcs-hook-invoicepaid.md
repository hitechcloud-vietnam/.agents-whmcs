# WHMCS InvoicePaid Hook Reference

## Overview

The `InvoicePaid` hook fires when an invoice is marked as paid. This is one of the most important hooks for payment processing, automation, and service fulfillment. It triggers when payment is received and the invoice status changes to "Paid".

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `invoiceid` | int | The unique invoice ID |
| `userid` | int | The client ID |
| `total` | float | Invoice total amount |
| `status` | string | Invoice status (Paid) |
| `paymentmethod` | string | Payment gateway used |
| `transid` | string | Transaction ID from payment gateway |
| `date` | string | Payment date |
| `model` | object | The Invoice model instance |

## Example Implementation

```php
<?php
add_hook('InvoicePaid', 1, function(array $params) {
    // Log the payment
    logActivity("Invoice #{$params['invoiceid']} paid: {$params['total']} via {$params['paymentmethod']}");
    
    // Get invoice details for further processing
    $invoice =WHMCS\Billing\Invoice::find($params['invoiceid']);
    $invoiceItems = $invoice->lineItems;
    
    foreach ($invoiceItems as $item) {
        if ($item->type === 'Hosting') {
            processHostingPayment($item->relid);
        } elseif ($item->type === 'Domain') {
            processDomainPayment($item->relid);
        }
    }
    
    return $params;
});
```

## Fulfilling Orders

```php
<?php
add_hook('InvoicePaid', 1, function(array $params) {
    // Get all items on the invoice
    $result = full_query("
        SELECT * FROM tblinvoiceitems 
        WHERE invoiceid = " . (int)$params['invoiceid']
    );
    
    while ($item = mysql_fetch_array($result)) {
        switch ($item['type']) {
            case 'Hosting':
            case 'Server':
                // Provision or activate service
                $service = Service::find($item['relid']);
                if ($service && $service->status !== 'Active') {
                    $service->status = 'Active';
                    $service->save();
                    // Optionally call module activate
                }
                break;
                
            case 'Domain':
                // Activate domain
                update_query('tbldomains', [
                    'status' => 'Active'
                ], ['id' => $item['relid']]);
                break;
                
            case 'Addon':
                // Activate addon
                update_query('tblhostingaddons', [
                    'status' => 'Active'
                ], ['id' => $item['relid']]);
                break;
        }
    }
    
    return $params;
});
```

## Payment Integration

```php
<?php
add_hook('InvoicePaid', 1, function(array $params) {
    // 1. Record in accounting system
    syncToAccountingSoftware($params);
    
    // 2. Update affiliate commissions
    updateAffiliateCommission($params['userid'], $params['total']);
    
    // 3. Send receipt to customer
    sendPaymentReceipt($params['invoiceid'], $params['userid']);
    
    // 4. Trigger fulfillment webhook
    sendWebhook('invoice.paid', [
        'invoice_id' => $params['invoiceid'],
        'amount' => $params['total'],
        'currency' => 'USD',
        'payment_method' => $params['paymentmethod'],
        'transaction_id' => $params['transid'],
        'customer_id' => $params['userid']
    ]);
    
    // 5. Update CRM
    updateCRMPaymentStatus($params['userid'], $params['invoiceid'], 'paid');
    
    return $params;
});
```

## Handling Partial Payments

```php
<?php
add_hook('InvoicePaid', 1, function(array $params) {
    $invoice = Invoice::find($params['invoiceid']);
    $total = $invoice->total;
    $paid = $invoice->amountPaid;
    
    // Check if fully paid
    if ($paid >= $total) {
        // Full payment - normal fulfillment
        fulfillInvoice($params['invoiceid']);
        
        // Send thank you email
        $client = getClientsDetails($params['userid']);
        sendTemplatedEmail('Payment Received', $client['email'], [
            'invoice_id' => $params['invoiceid'],
            'amount' => $params['total']
        ]);
    } else {
        // Partial payment - log and handle differently
        logActivity("Partial payment received for invoice #{$params['invoiceid']}: {$paid} of {$total}");
        
        // Apply to oldest line items first
        applyPartialPayment($params['invoiceid'], $paid);
    }
    
    return $params;
});
```

## Use Cases

- **Service Fulfillment**: Activate services when paid
- **Accounting Integration**: Sync payments to accounting software
- **Affiliate Commissions**: Calculate and credit referrals
- **Webhooks**: Notify external systems
- **Receipt Generation**: Send payment confirmations

## Notes

- This hook fires when status changes to "Paid"
- Multiple payments on a single invoice may trigger this multiple times
- Use with `InvoiceCreated` for complete invoice lifecycle
- Check `transid` for duplicate payment prevention

## Related Hooks

- `InvoiceCreated` - When invoice is created
- `InvoiceCancelled` - When invoice is cancelled
- `OrderPaid` - When an order is paid

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Payment Gateway Development](../whmcs-gateway-development.md)