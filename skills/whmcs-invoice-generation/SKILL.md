# WHMCS Invoice Generation

## Overview
Master skill for invoice generation in WHMCS. Covers invoice creation, line items, tax calculation, and customization.

## Invoice Generation Hooks

```php
<?php
// /includes/hooks/invoice_hooks.php

/**
 * Before invoice creation
 */
add_hook('PreInvoiceCreation', 1, function(array $params) {
    $clientId = $params['userid'];
    $client = \WHMCS\User\Client::find($clientId);

    // Check for overdue invoices
    $hasOverdue = $client->invoices()
        ->where('status', 'Unpaid')
        ->where('duedate', '<', date('Y-m-d'))
        ->exists();

    if ($hasOverdue) {
        // Add late fee
        $params['lineitems'][] = [
            'description' => 'Late Payment Fee',
            'amount' => 5.00,
            'taxed' => false,
        ];
    }

    return $params;
});

/**
 * After invoice creation
 */
add_hook('InvoiceCreated', 1, function(array $params) {
    $invoiceId = $params['invoiceid'];
    $invoice = \WHMCS\Billing\Invoice::find($invoiceId);

    // Send invoice notification
    send_email('InvoiceCreated', $invoice->clientId, [
        'invoice_id' => $invoiceId,
    ]);

    // Update accounting system
    syncInvoiceToAccounting($invoice);

    return true;
});

/**
 * Invoice creation - add custom items
 */
add_hook('InvoiceCreation', 1, function(array $params) {
    $invoiceId = $params['invoiceid'];

    // Add handling fee for specific payment methods
    $paymentMethod = $params['paymentmethod'] ?? '';

    if ($paymentMethod === 'bank_transfer') {
        addInvoiceLineItem($invoiceId, [
            'description' => 'Bank Transfer Processing Fee',
            'amount' => 2.50,
            'taxed' => false,
        ]);
    }

    return true;
});

/**
 * Modify invoice line items
 */
add_hook('InvoiceLineItem', 1, function(array $params) {
    $item = $params['item'];

    // Add description prefix for recurring items
    if ($item['type'] === 'recurring') {
        $item['description'] = 'Recurring: ' . $item['description'];
    }

    return $item;
});

/**
 * Before invoice deletion
 */
add_hook('PreInvoiceDeletion', 1, function(array $params) {
    $invoiceId = $params['invoiceid'];
    $invoice = \WHMCS\Billing\Invoice::find($invoiceId);

    if ($invoice && $invoice->status === 'Paid') {
        return [
            'error' => 'Cannot delete a paid invoice',
        ];
    }

    // Remove from external systems
    removeInvoiceFromAccounting($invoiceId);

    return true;
});
```

## Invoice Helper Functions

```php
<?php
// /includes/helpers/invoice_helper.php

/**
 * Create invoice for client
 */
function createInvoice(array $params): int
{
    $clientId = $params['client_id'];
    $dueDate = $params['due_date'] ?? date('Y-m-d', strtotime('+7 days'));
    $paymentMethod = $params['payment_method'] ?? 'gateway';

    $invoiceId = localapi('CreateInvoice', [
        'userid' => $clientId,
        'duedate' => $dueDate,
        'paymentmethod' => $paymentMethod,
    ]);

    return $invoiceId;
}

/**
 * Add line item to invoice
 */
function addInvoiceLineItem(int $invoiceId, array $item): bool
{
    $itemId = \Illuminate\Database\Capsule\Manager::table('tblinvoiceitems')
        ->insertGetId([
            'invoiceid' => $invoiceId,
            'userid' => getInvoiceClientId($invoiceId),
            'type' => $item['type'] ?? 'Item',
            'relid' => $item['rel_id'] ?? 0,
            'description' => $item['description'],
            'amount' => $item['amount'],
            'taxed' => $item['taxed'] ?? true,
            'sort_order' => getNextInvoiceItemSort($invoiceId),
        ]);

    // Recalculate invoice totals
    recalculateInvoice($invoiceId);

    return $itemId > 0;
}

/**
 * Remove line item from invoice
 */
function removeInvoiceLineItem(int $itemId): bool
{
    $item = \Illuminate\Database\Capsule\Manager::table('tblinvoiceitems')
        ->find($itemId);

    if (!$item) {
        return false;
    }

    $invoiceId = $item->invoiceid;

    \Illuminate\Database\Capsule\Manager::table('tblinvoiceitems')
        ->where('id', $itemId)
        ->delete();

    // Recalculate invoice totals
    recalculateInvoice($invoiceId);

    return true;
}

/**
 * Recalculate invoice totals
 */
function recalculateInvoice(int $invoiceId): void
{
    $invoice = \WHMCS\Billing\Invoice::find($invoiceId);

    if (!$invoice) {
        return;
    }

    $items = $invoice->lineItems;
    $subtotal = 0;
    $tax = 0;
    $tax2 = 0;

    $client = $invoice->client;
    $taxRate = getClientTaxRate($client);
    $taxRate2 = getClientTaxRate2($client);

    foreach ($items as $item) {
        $subtotal += $item->amount;

        if ($item->taxed) {
            $tax += $item->amount * ($taxRate / 100);

            if ($taxRate2 > 0) {
                $tax2 += ($item->amount + ($item->amount * $taxRate / 100)) * ($taxRate2 / 100);
            }
        }
    }

    $total = $subtotal + $tax + $tax2;

    $invoice->subtotal = $subtotal;
    $invoice->tax = $tax;
    $invoice->tax2 = $tax2;
    $invoice->total = $total;
    $invoice->taxrate = $taxRate;
    $invoice->taxrate2 = $taxRate2;
    $invoice->save();
}

/**
 * Get client tax rate
 */
function getClientTaxRate(\WHMCS\User\Client $client): float
{
    // Check if client is tax exempt
    if ($client->taxexempt) {
        return 0;
    }

    // Get tax rate based on state/country
    $taxRule = \Illuminate\Database\Capsule\Manager::table('tbltax')
        ->where('country', $client->country)
        ->where(function($q) use ($client) {
            $q->where('state', $client->state)
                ->orWhere('state', '');
        })
        ->where('level', 'like', '%' . ($client->isIndividual() ? 'individual' : 'business') . '%')
        ->first();

    return $taxRule ? (float)$taxRule->taxrate : 0;
}

/**
 * Generate invoice number
 */
function generateInvoiceNumber(string $prefix = 'INV'): string
{
    $year = date('Y');
    $sequence = getNextInvoiceSequence($year);

    return sprintf('%s-%s-%05d', $prefix, $year, $sequence);
}

/**
 * Get next invoice sequence number
 */
function getNextInvoiceSequence(int $year): int
{
    $lastInvoice = \Illuminate\Database\Capsule\Manager::table('tblinvoices')
        ->where('invoicenum', 'like', '%' . $year . '%')
        ->orderBy('id', 'desc')
        ->first();

    if ($lastInvoice && preg_match('/(\d+)$/', $lastInvoice->invoicenum, $matches)) {
        return (int)$matches[1] + 1;
    }

    return 1;
}

/**
 * Create invoice from order
 */
function createInvoiceFromOrder(int $orderId): int
{
    $order = \WHMCS\Order\Order::find($orderId);

    $invoiceId = localapi('CreateInvoice', [
        'userid' => $order->userid,
        'draft' => false,
    ]);

    // Add order items to invoice
    foreach ($order->lineItems as $item) {
        addInvoiceLineItem($invoiceId, [
            'description' => $item->description,
            'amount' => $item->amount,
            'taxed' => true,
        ]);
    }

    return $invoiceId;
}

/**
 * Clone invoice
 */
function cloneInvoice(int $invoiceId): int
{
    $original = \WHMCS\Billing\Invoice::find($invoiceId);

    if (!$original) {
        return 0;
    }

    $newInvoiceId = localapi('CreateInvoice', [
        'userid' => $original->clientId,
        'duedate' => date('Y-m-d', strtotime('+30 days')),
        'paymentmethod' => $original->paymentmethod,
    ]);

    // Copy line items
    foreach ($original->lineItems as $item) {
        addInvoiceLineItem($newInvoiceId, [
            'description' => $item->description,
            'amount' => $item->amount,
            'taxed' => $item->taxed,
        ]);
    }

    return $newInvoiceId;
}

/**
 * Split invoice items
 */
function splitInvoice(int $invoiceId, array $itemIds): int
{
    $original = \WHMCS\Billing\Invoice::find($invoiceId);

    if (!$original) {
        return 0;
    }

    // Create new invoice
    $newInvoiceId = localapi('CreateInvoice', [
        'userid' => $original->clientId,
        'duedate' => $original->duedate,
        'paymentmethod' => $original->paymentmethod,
    ]);

    // Move selected items to new invoice
    foreach ($itemIds as $itemId) {
        $item = \Illuminate\Database\Capsule\Manager::table('tblinvoiceitems')
            ->find($itemId);

        if ($item && $item->invoiceid == $invoiceId) {
            \Illuminate\Database\Capsule\Manager::table('tblinvoiceitems')
                ->where('id', $itemId)
                ->update(['invoiceid' => $newInvoiceId]);
        }
    }

    // Recalculate both invoices
    recalculateInvoice($invoiceId);
    recalculateInvoice($newInvoiceId);

    return $newInvoiceId;
}

/**
 * Merge invoices
 */
function mergeInvoices(array $invoiceIds): int
{
    if (count($invoiceIds) < 2) {
        return 0;
    }

    $firstInvoice = \WHMCS\Billing\Invoice::find($invoiceIds[0]);

    if (!$firstInvoice) {
        return 0;
    }

    // Move all items from other invoices to first
    foreach (array_slice($invoiceIds, 1) as $invoiceId) {
        $items = \Illuminate\Database\Capsule\Manager::table('tblinvoiceitems')
            ->where('invoiceid', $invoiceId)
            ->get();

        foreach ($items as $item) {
            \Illuminate\Database\Capsule\Manager::table('tblinvoiceitems')
                ->where('id', $item->id)
                ->update(['invoiceid' => $firstInvoice->id]);
        }

        // Delete empty invoice
        \Illuminate\Database\Capsule\Manager::table('tblinvoices')
            ->where('id', $invoiceId)
            ->delete();
    }

    // Recalculate first invoice
    recalculateInvoice($firstInvoice->id);

    return $firstInvoice->id;
}
```

## Custom Invoice Template

```smarty
{* /templates/your_template/invoice.tpl *}
{extends file="$layout"}

{block name="content"}
<div class="invoice" id="invoice">
    <div class="invoice-header">
        <div class="row">
            <div class="col-md-6">
                <h2>{$companyname}</h2>
                <p>{$address1}<br>
                {if $address2}{$address2}<br>{/if}
                {$city}, {$state} {$postcode}<br>
                {$country}</p>
                <p>{$companyemail}</p>
            </div>
            <div class="col-md-6 text-right">
                <h1>INVOICE</h1>
                <p><strong>Invoice #:</strong> {$invoicenum}</p>
                <p><strong>Date:</strong> {$date}</p>
                <p><strong>Due Date:</strong> {$duedate}</p>
                <p><strong>Status:</strong> <span class="badge badge-{$status_class}">{$status}</span></p>
            </div>
        </div>
    </div>

    <div class="invoice-bill-to mt-4">
        <h5>Bill To:</h5>
        <p>
            <strong>{$clientname}</strong><br>
            {if $clientcompany}{{$clientcompany}}<br>{/if}
            {$clientaddress1}<br>
            {if $clientaddress2}{$clientaddress2}<br>{/if}
            {$clientcity}, {$clientstate} {$clientpostcode}<br>
            {$clientcountry}<br>
            {$clientemail}
        </p>
    </div>

    <div class="invoice-items mt-4">
        <table class="table table-bordered">
            <thead class="thead-light">
                <tr>
                    <th>Description</th>
                    <th class="text-right">Amount</th>
                </tr>
            </thead>
            <tbody>
                {foreach $lineitems as $item}
                <tr>
                    <td>
                        {$item.description}
                        {if $item.type === 'recurring'}
                        <br><small class="text-muted">({$item.billingcycle})</small>
                        {/if}
                    </td>
                    <td class="text-right">{$item.amount}</td>
                </tr>
                {/foreach}
            </tbody>
        </table>
    </div>

    <div class="invoice-totals mt-4">
        <table class="table table-sm ml-auto" style="max-width: 300px;">
            <tr>
                <td class="text-right"><strong>Subtotal:</strong></td>
                <td class="text-right">{$subtotal}</td>
            </tr>
            {if $taxrate > 0}
            <tr>
                <td class="text-right">Tax ({$taxrate}%):</td>
                <td class="text-right">{$tax}</td>
            </tr>
            {/if}
            {if $taxrate2 > 0}
            <tr>
                <td class="text-right">Tax 2 ({$taxrate2}%):</td>
                <td class="text-right">{$tax2}</td>
            </tr>
            {/if}
            <tr class="table-active">
                <td class="text-right"><strong>Total:</strong></td>
                <td class="text-right"><strong>{$total}</strong></td>
            </tr>
            {if $creditapplied > 0}
            <tr>
                <td class="text-right">Credit Applied:</td>
                <td class="text-right">-{$creditapplied}</td>
            </tr>
            <tr class="table-success">
                <td class="text-right"><strong>Balance Due:</strong></td>
                <td class="text-right"><strong>{$balance}</strong></td>
            </tr>
            {/if}
        </table>
    </div>

    {if $status === 'Unpaid'}
    <div class="invoice-payment mt-4">
        <h5>Payment Methods</h5>
        <div class="row">
            {foreach $paymentmethods as $method}
            <div class="col-md-3">
                <a href="{$systemurl}dopayment.php?invoiceid={$invoiceid}&paymentmethod={$method.sysname}" 
                   class="btn btn-outline-primary btn-block">
                    {$method.name}
                </a>
            </div>
            {/foreach}
        </div>
    </div>
    {/if}

    {if $notes}
    <div class="invoice-notes mt-4">
        <h5>Notes</h5>
        <p>{$notes}</p>
    </div>
    {/if}
</div>

<div class="invoice-actions mt-4">
    <a href="{$systemurl}viewinvoice.php?id={$invoiceid}&print=true" class="btn btn-secondary" target="_blank">
        <i class="fa fa-print"></i> Print Invoice
    </a>
    <a href="{$systemurl}dl.php?id={$invoiceid}" class="btn btn-secondary">
        <i class="fa fa-download"></i> Download PDF
    </a>
</div>
{/block}
```

## Best Practices

1. **Unique Invoice Numbers**: Always generate unique invoice numbers
2. **Tax Calculation**: Properly calculate and display taxes
3. **Line Item Details**: Include all necessary details for each item
4. **Due Dates**: Set clear and reasonable due dates
5. **Payment Options**: Provide multiple payment methods
6. **Invoice Status**: Keep invoice status accurate
7. **Partial Payments**: Support partial payments
8. **Credits**: Apply client credits correctly
9. **Templates**: Use professional invoice templates
10. **Notifications**: Send reminders for unpaid invoices
