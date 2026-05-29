# WHMCS Invoice Functions

Complete reference for managing invoices in WHMCS.

## Overview

WHMCS provides comprehensive invoice management functions for creating, updating, and managing invoices throughout their lifecycle.

## Core Invoice Functions

### createInvoice()

Creates a new invoice.

```php
/**
 * Create a new invoice
 * 
 * @param array $data Invoice data
 * @return int Invoice ID
 */
function createInvoice(array $data): int
{
    return Capsule::table('tblinvoices')->insertGetId([
        'userid' => $data['clientid'] ?? 0,
        'invoicenum' => $data['invoicenum'] ?? '',
        'date' => $data['date'] ?? date('Y-m-d'),
        'duedate' => $data['duedate'] ?? date('Y-m-d', strtotime('+7 days')),
        'datepaid' => $data['datepaid'] ?? null,
        'datecleared' => $data['datecleared'] ?? null,
        'total' => $data['total'] ?? 0,
        'taxrate' => $data['taxrate'] ?? 0,
        'taxrate2' => $data['taxrate2'] ?? 0,
        'subtotal' => $data['subtotal'] ?? 0,
        'credit' => $data['credit'] ?? 0,
        'tax' => $data['tax'] ?? 0,
        'tax2' => $data['tax2'] ?? 0,
        'status' => $data['status'] ?? 'Draft',
        'paymentmethod' => $data['paymentmethod'] ?? '',
        'notes' => $data['notes'] ?? '',
        'currency' => $data['currency'] ?? 0,
        'prjid' => $data['projectid'] ?? 0,
    ]);
}
```

**Example:**
```php
$invoiceId = createInvoice([
    'clientid' => 123,
    'invoicenum' => 'INV-2024-0001',
    'date' => date('Y-m-d'),
    'duedate' => date('Y-m-d', strtotime('+14 days')),
    'subtotal' => 100.00,
    'tax' => 10.00,
    'total' => 110.00,
    'status' => 'Unpaid',
    'paymentmethod' => 'stripe'
]);
```

### getInvoice()

Retrieves an invoice by ID.

```php
/**
 * Get invoice by ID
 * 
 * @param int $invoiceId Invoice ID
 * @return array|null Invoice data
 */
function getInvoice(int $invoiceId): ?array
{
    $result = Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->first();
    
    return $result ? (array) $result : null;
}
```

**Example:**
```php
$invoice = getInvoice(1001);

if ($invoice) {
    echo "Invoice #{$invoice['invoicenum']} - {$invoice['total']}";
}
```

### updateInvoice()

Updates an existing invoice.

```php
/**
 * Update an invoice
 * 
 * @param int $invoiceId Invoice ID
 * @param array $data Updated data
 * @return bool Success status
 */
function updateInvoice(int $invoiceId, array $data): bool
{
    return Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->update($data);
}
```

**Example:**
```php
updateInvoice(1001, [
    'duedate' => date('Y-m-d', strtotime('+30 days')),
    'notes' => 'Extended payment terms - approved by manager'
]);
```

### deleteInvoice()

Deletes an invoice.

```php
/**
 * Delete an invoice
 * 
 * @param int $invoiceId Invoice ID
 * @return bool Success status
 */
function deleteInvoice(int $invoiceId): bool
{
    // Verify invoice can be deleted
    $invoice = getInvoice($invoiceId);
    
    if ($invoice && $invoice['status'] === 'Paid') {
        return false; // Cannot delete paid invoices
    }
    
    // Delete associated items
    Capsule::table('tblinvoiceitems')
        ->where('invoiceid', $invoiceId)
        ->delete();
    
    // Delete invoice
    return Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->delete() > 0;
}
```

## Invoice Items

### addInvoiceItem()

Adds an item to an invoice.

```php
/**
 * Add item to invoice
 * 
 * @param array $data Item data
 * @return int Item ID
 */
function addInvoiceItem(array $data): int
{
    return Capsule::table('tblinvoiceitems')->insertGetId([
        'invoiceid' => $data['invoiceid'],
        'userid' => $data['clientid'] ?? 0,
        'type' => $data['type'] ?? 'Hosting',
        'relid' => $data['relid'] ?? 0,
        'description' => $data['description'] ?? '',
        'amount' => $data['amount'] ?? 0,
        'taxed' => $data['taxed'] ?? 0,
        'qty' => $data['qty'] ?? 1,
        'duedate' => $data['duedate'] ?? null,
    ]);
}
```

**Example:**
```php
addInvoiceItem([
    'invoiceid' => 1001,
    'clientid' => 123,
    'type' => 'Hosting',
    'relid' => 456,
    'description' => 'Premium Hosting Plan - Monthly',
    'amount' => 9.99,
    'taxed' => 1,
    'qty' => 1,
    'duedate' => date('Y-m-d')
]);

addInvoiceItem([
    'invoiceid' => 1001,
    'clientid' => 123,
    'type' => 'Domain',
    'relid' => 789,
    'description' => 'Domain Renewal - example.com (1 year)',
    'amount' => 15.99,
    'taxed' => 0,
    'qty' => 1
]);
```

### getInvoiceItems()

Retrieves all items for an invoice.

```php
/**
 * Get invoice items
 * 
 * @param int $invoiceId Invoice ID
 * @return array Items
 */
function getInvoiceItems(int $invoiceId): array
{
    return Capsule::table('tblinvoiceitems')
        ->where('invoiceid', $invoiceId)
        ->get()
        ->toArray();
}
```

## Invoice Operations

### addInvoicePayment()

Records a payment against an invoice.

```php
/**
 * Add payment to invoice
 * 
 * @param int $invoiceId Invoice ID
 * @param string $transactionId Transaction ID
 * @param float $amount Payment amount
 * @param string $date Payment date
 * @return bool Success status
 */
function addInvoicePayment(
    int $invoiceId,
    string $transactionId,
    float $amount,
    string $date = ''
): bool {
    $date = $date ?: date('Y-m-d H:i:s');
    
    Capsule::table('tblinvoicepayments')->insert([
        'invoice_id' => $invoiceId,
        'trans_id' => $transactionId,
        'amount' => $amount,
        'date' => $date
    ]);
    
    // Update invoice status
    $invoice = getInvoice($invoiceId);
    $payments = getInvoicePaymentsTotal($invoiceId);
    
    $newStatus = 'Unpaid';
    if ($payments >= $invoice['total']) {
        $newStatus = 'Paid';
    } elseif ($payments > 0) {
        $newStatus = 'Partial';
    }
    
    updateInvoice($invoiceId, [
        'status' => $newStatus,
        'datepaid' => $newStatus === 'Paid' ? $date : null
    ]);
    
    return true;
}
```

### getInvoicePaymentsTotal()

Gets total payments for an invoice.

```php
/**
 * Get total payments for invoice
 * 
 * @param int $invoiceId Invoice ID
 * @return float Total paid
 */
function getInvoicePaymentsTotal(int $invoiceId): float
{
    return (float) Capsule::table('tblinvoicepayments')
        ->where('invoice_id', $invoiceId)
        ->sum('amount');
}
```

### addInvoiceCredit()

Applies client credit to an invoice.

```php
/**
 * Apply credit to invoice
 * 
 * @param int $invoiceId Invoice ID
 * @param float $amount Credit amount
 * @param string $description Description
 * @return bool Success status
 */
function addInvoiceCredit(
    int $invoiceId,
    float $amount,
    string $description = 'Credit applied'
): bool {
    $invoice = getInvoice($invoiceId);
    
    if (!$invoice) {
        return false;
    }
    
    // Update invoice credit
    updateInvoice($invoiceId, [
        'credit' => $invoice['credit'] + $amount
    ]);
    
    // Deduct from client balance
    Capsule::table('tblclients')
        ->where('id', $invoice['userid'])
        ->decrement('credit', $amount);
    
    // Log transaction
    addTransaction([
        'clientid' => $invoice['userid'],
        'description' => $description,
        'amountout' => $amount,
        'invoiceid' => $invoiceId
    ]);
    
    return true;
}
```

## Invoice Generation

### generateInvoice()

Generates a complete invoice with line items.

```php
/**
 * Generate invoice from order
 * 
 * @param int $orderId Order ID
 * @param bool $sendEmail Send notification
 * @return int Invoice ID
 */
function generateInvoice(int $orderId, bool $sendEmail = true): int
{
    $order = Capsule::table('tblorders')
        ->where('id', $orderId)
        ->first();
    
    if (!$order) {
        throw new Exception('Order not found');
    }
    
    // Create invoice
    $invoiceId = createInvoice([
        'clientid' => $order->userid,
        'date' => date('Y-m-d'),
        'duedate' => date('Y-m-d', strtotime('+7 days')),
        'subtotal' => $order->amount,
        'total' => $order->amount + ($order->amount * 0.1), // Add tax
        'status' => 'Unpaid',
        'paymentmethod' => $order->paymentmethod
    ]);
    
    // Add order items
    $orderItems = Capsule::table('tblorderitems')
        ->where('orderid', $orderId)
        ->get();
    
    foreach ($orderItems as $item) {
        addInvoiceItem([
            'invoiceid' => $invoiceId,
            'clientid' => $order->userid,
            'type' => $item->type,
            'relid' => $item->id,
            'description' => $item->description,
            'amount' => $item->amount,
            'qty' => $item->qty
        ]);
    }
    
    // Update order with invoice
    Capsule::table('tblorders')
        ->where('id', $orderId)
        ->update(['invoiceid' => $invoiceId]);
    
    return $invoiceId;
}
```

## Invoice Status Management

```php
/**
 * Update invoice status based on payments
 * 
 * @param int $invoiceId Invoice ID
 * @return string New status
 */
function recalculateInvoiceStatus(int $invoiceId): string
{
    $invoice = getInvoice($invoiceId);
    $totalPaid = getInvoicePaymentsTotal($invoiceId);
    $creditApplied = $invoice['credit'] ?? 0;
    $totalDue = $invoice['total'] ?? 0;
    
    $amountApplied = $totalPaid + $creditApplied;
    
    if ($amountApplied >= $totalDue) {
        $newStatus = 'Paid';
        Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->update([
                'status' => 'Paid',
                'datepaid' => date('Y-m-d H:i:s')
            ]);
    } elseif ($amountApplied > 0) {
        $newStatus = 'Partial';
        Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->update(['status' => 'Partial']);
    } else {
        $newStatus = 'Unpaid';
    }
    
    return $newStatus;
}
```

## Invoice Queries

### getInvoices()

Retrieves multiple invoices with filtering.

```php
/**
 * Get invoices with filters
 * 
 * @param array $filters Filter options
 * @param int $limit Number of records
 * @param int $offset Starting offset
 * @return array Invoices
 */
function getInvoices(array $filters = [], int $limit = 50, int $offset = 0): array
{
    $query = Capsule::table('tblinvoices')
        ->select('tblinvoices.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tblinvoices.userid')
        ->orderBy('tblinvoices.id', 'desc');
    
    if (!empty($filters['clientId'])) {
        $query->where('tblinvoices.userid', $filters['clientId']);
    }
    
    if (!empty($filters['status'])) {
        $query->where('tblinvoices.status', $filters['status']);
    }
    
    if (!empty($filters['dateFrom'])) {
        $query->where('tblinvoices.date', '>=', $filters['dateFrom']);
    }
    
    if (!empty($filters['dateTo'])) {
        $query->where('tblinvoices.date', '<=', $filters['dateTo']);
    }
    
    if (!empty($filters['overdue'])) {
        $query->where('tblinvoices.duedate', '<', date('Y-m-d'))
            ->where('tblinvoices.status', 'Unpaid');
    }
    
    return $query->limit($limit)
        ->offset($offset)
        ->get()
        ->toArray();
}
```

**Example:**
```php
// Get all overdue invoices
$overdue = getInvoices(['overdue' => true]);

// Get client's paid invoices
$paid = getInvoices([
    'clientId' => 123,
    'status' => 'Paid'
]);
```

## Invoice PDF Generation

```php
/**
 * Generate invoice PDF
 * 
 * @param int $invoiceId Invoice ID
 * @return string PDF content
 */
function generateInvoicePDF(int $invoiceId): string
{
    $invoice = getInvoice($invoiceId);
    $items = getInvoiceItems($invoiceId);
    $client = Capsule::table('tblclients')
        ->where('id', $invoice['userid'])
        ->first();
    
    // Build PDF using WHMCS template system
    $pdf = new WHMCS\Invoice\PDF();
    $pdf->setInvoiceData($invoice, $items, $client);
    
    return $pdf->generate();
}
```

## Automatic Invoice Operations

```php
/**
 * Process overdue invoices - run via cron
 */
function processOverdueInvoices(): array
{
    $overdueInvoices = getInvoices([
        'status' => 'Unpaid',
        'overdue' => true
    ], 1000);
    
    $processed = 0;
    
    foreach ($overdueInvoices as $invoice) {
        // Send reminder if not sent recently
        $lastReminder = getLastReminderDate($invoice['id']);
        
        if (!$lastReminder || strtotime($lastReminder) < strtotime('-7 days')) {
            sendInvoiceReminder($invoice['id']);
            $processed++;
        }
    }
    
    return ['processed' => $processed];
}

/**
 * Create overdue notices
 */
function createOverdueNotices(): array
{
    $invoices = getInvoices([
        'overdue' => true,
        'status' => 'Unpaid'
    ]);
    
    $notices = [];
    
    foreach ($invoices as $invoice) {
        $notices[] = [
            'invoice_id' => $invoice['id'],
            'client_id' => $invoice['userid'],
            'amount' => $invoice['total'],
            'days_overdue' => daysOverdue($invoice['duedate'])
        ];
    }
    
    return $notices;
}
```

## Recurring Invoices

```php
/**
 * Create next invoice from recurring
 * 
 * @param int $serviceId Service ID
 * @return int|null New invoice ID or null
 */
function createNextRecurringInvoice(int $serviceId): ?int
{
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();
    
    if (!$service || !$service->nextinvoicedate) {
        return null;
    }
    
    // Check if it's time to generate
    if (strtotime($service->nextinvoicedate) > time()) {
        return null;
    }
    
    // Create invoice
    $invoiceId = createInvoice([
        'clientid' => $service->userid,
        'date' => date('Y-m-d'),
        'duedate' => date('Y-m-d', strtotime('+7 days')),
        'subtotal' => $service->amount,
        'total' => $service->amount,
        'status' => 'Unpaid'
    ]);
    
    // Add service item
    addInvoiceItem([
        'invoiceid' => $invoiceId,
        'clientid' => $service->userid,
        'type' => 'Hosting',
        'relid' => $service->id,
        'description' => $service->domain'] . ' - ' . date('M Y'),
        'amount' => $service->amount,
        'duedate' => date('Y-m-d')
    ]);
    
    // Update next invoice date
    $nextDate = calculateNextBillingDate(
        $service->nextinvoicedate,
        $service->billingcycle
    );
    
    Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->update(['nextinvoicedate' => $nextDate]);
    
    return $invoiceId;
}
```

## Best Practices

1. **Always recalculate totals** - Update invoice totals after adding/removing items
2. **Maintain audit trail** - Log all invoice changes
3. **Use proper status transitions** - Follow invoice lifecycle states
4. **Handle partial payments** - Track payment progress accurately
5. **Send notifications** - Email clients on status changes
6. **Archive old invoices** - Move paid invoices to archive regularly

## Related Functions

- [whmcs-functions-transactions.md](whmcs-functions-transactions.md) - Payment handling
- [whmcs-functions-clients.md](whmcs-functions-clients.md) - Client operations
- [whmcs-schema-invoices.md](whmcs-schema-invoices.md) - Invoice database schema