# WHMCS Transaction Functions

Complete reference for handling financial transactions in WHMCS.

## Overview

WHMCS provides a comprehensive set of functions for managing transactions including payments, refunds, credits, and ledger operations.

## Transaction Functions

### addTransaction()

Adds a transaction to the system.

```php
/**
 * Add a new transaction
 * 
 * @param array $data Transaction data
 * @return int Transaction ID
 */
function addTransaction(array $data): int
{
    return Capsule::table('tbltransections')->insertGetId([
        'userid' => $data['clientid'] ?? 0,
        'currency' => $data['currency'] ?? 0,
        'date' => $data['date'] ?? date('Y-m-d H:i:s'),
        'description' => $data['description'] ?? '',
        'amount' => $data['amount'] ?? 0,
        'amountin' => $data['amountin'] ?? 0,
        'amountout' => $data['amountout'] ?? 0,
        'rate' => $data['rate'] ?? 1,
        'transid' => $data['transid'] ?? '',
        'invoiceid' => $data['invoiceid'] ?? 0,
        'refundid' => $data['refundid'] ?? 0,
        'gateway' => $data['gateway'] ?? '',
        'fee' => $data['fee'] ?? 0,
    ]);
}
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| clientid | int | Client ID |
| currency | int | Currency ID |
| date | string | Transaction date (Y-m-d H:i:s) |
| description | string | Transaction description |
| amount | float | Transaction amount |
| amountin | float | Amount credited |
| amountout | float | Amount debited |
| transid | string | External transaction ID |
| invoiceid | int | Associated invoice ID |
| gateway | string | Payment gateway name |
| fee | float | Transaction fee |

**Example:**
```php
$result = addTransaction([
    'clientid' => 123,
    'description' => 'Payment for Invoice #1001',
    'amountin' => 99.99,
    'gateway' => 'stripe',
    'transid' => 'ch_1234567890',
    'invoiceid' => 1001
]);

logActivity("Transaction #{$result} added", 0);
```

### updateTransaction()

Updates an existing transaction.

```php
/**
 * Update a transaction
 * 
 * @param int $transactionId Transaction ID
 * @param array $data Updated data
 * @return bool Success status
 */
function updateTransaction(int $transactionId, array $data): bool
{
    $data['id'] = $transactionId;
    $data['updated_at'] = date('Y-m-d H:i:s');
    
    return Capsule::table('tbltransections')
        ->where('id', $transactionId)
        ->update($data);
}
```

**Example:**
```php
updateTransaction(456, [
    'description' => 'Updated - Partial payment received',
    'amountout' => 25.00
]);
```

### getTransaction()

Retrieves a single transaction by ID.

```php
/**
 * Get a transaction by ID
 * 
 * @param int $transactionId Transaction ID
 * @return array|null Transaction data or null
 */
function getTransaction(int $transactionId): ?array
{
    $result = Capsule::table('tbltransections')
        ->where('id', $transactionId)
        ->first();
    
    return $result ? (array) $result : null;
}
```

**Example:**
```php
$transaction = getTransaction(456);
if ($transaction) {
    echo "Amount: {$transaction['amount']}";
}
```

### getTransactions()

Retrieves multiple transactions with filtering.

```php
/**
 * Get transactions with filters
 * 
 * @param array $filters Filter options
 * @param int $limit Number of records
 * @param int $offset Starting offset
 * @return array Transactions
 */
function getTransactions(array $filters = [], int $limit = 50, int $offset = 0): array
{
    $query = Capsule::table('tbltransections')
        ->select('tbltransections.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tbltransections.userid')
        ->orderBy('tbltransections.id', 'desc');
    
    if (!empty($filters['clientId'])) {
        $query->where('tbltransections.userid', $filters['clientId']);
    }
    
    if (!empty($filters['invoiceId'])) {
        $query->where('tbltransections.invoiceid', $filters['invoiceId']);
    }
    
    if (!empty($filters['gateway'])) {
        $query->where('tbltransections.gateway', $filters['gateway']);
    }
    
    if (!empty($filters['dateFrom'])) {
        $query->where('tbltransections.date', '>=', $filters['dateFrom']);
    }
    
    if (!empty($filters['dateTo'])) {
        $query->where('tbltransections.date', '<=', $filters['dateTo']);
    }
    
    return $query->limit($limit)
        ->offset($offset)
        ->get()
        ->toArray();
}
```

**Example:**
```php
$transactions = getTransactions([
    'clientId' => 123,
    'gateway' => 'stripe',
    'dateFrom' => '2024-01-01',
    'dateTo' => '2024-12-31'
], 100, 0);
```

## Refund Functions

### refundTransaction()

Processes a refund for a transaction.

```php
/**
 * Process a refund
 * 
 * @param int $transactionId Original transaction ID
 * @param float $amount Refund amount
 * @param string $reason Refund reason
 * @param bool $sendEmail Send notification email
 * @return array Result with transaction ID
 */
function refundTransaction(
    int $transactionId,
    float $amount,
    string $reason = '',
    bool $sendEmail = true
): array {
    $original = getTransaction($transactionId);
    
    if (!$original) {
        return ['success' => false, 'error' => 'Transaction not found'];
    }
    
    // Create refund transaction
    $refundId = addTransaction([
        'clientid' => $original['userid'],
        'description' => "Refund: {$original['description']}",
        'amountout' => $amount,
        'gateway' => $original['gateway'],
        'transid' => 'REF-' . time(),
        'refundid' => $transactionId,
        'date' => date('Y-m-d H:i:s')
    ]);
    
    // Update original transaction
    updateTransaction($transactionId, [
        'refundid' => $refundId
    ]);
    
    // Create ticket comment if reason provided
    if ($reason) {
        logActivity("Refund processed: {$reason}", 0);
    }
    
    return [
        'success' => true,
        'refundId' => $refundId
    ];
}
```

**Example:**
```php
$result = refundTransaction(456, 49.99, 'Customer request - product not as described');

if ($result['success']) {
    echo "Refund processed: #{$result['refundId']}";
}
```

## Payment Application

### applyTransaction()

Applies a transaction to an invoice.

```php
/**
 * Apply a transaction to an invoice
 * 
 * @param int $transactionId Transaction ID
 * @param int $invoiceId Invoice ID
 * @return bool Success status
 */
function applyTransaction(int $transactionId, int $invoiceId): bool
{
    $transaction = getTransaction($transactionId);
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->first();
    
    if (!$transaction || !$invoice) {
        return false;
    }
    
    // Update transaction with invoice
    updateTransaction($transactionId, [
        'invoiceid' => $invoiceId
    ]);
    
    // Add invoice payment
    Capsule::table('tblinvoicepayments')->insert([
        'invoice_id' => $invoiceId,
        'trans_id' => $transactionId,
        'amount' => $transaction['amountin'],
        'date' => date('Y-m-d H:i:s')
    ]);
    
    // Update invoice status if paid
    $payments = Capsule::table('tblinvoicepayments')
        ->where('invoice_id', $invoiceId)
        ->sum('amount');
    
    if ($payments >= $invoice->total) {
        Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->update(['status' => 'Paid']);
    }
    
    return true;
}
```

**Example:**
```php
applyTransaction(456, 1001);
```

## Client Balance

### updateClientBalance()

Updates a client's credit balance.

```php
/**
 * Update client credit balance
 * 
 * @param int $clientId Client ID
 * @param float $amount Amount to add (negative to subtract)
 * @param string $description Description of change
 * @return bool Success status
 */
function updateClientBalance(int $clientId, float $amount, string $description = ''): bool
{
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    if (!$client) {
        return false;
    }
    
    $newBalance = ($client->credit ?? 0) + $amount;
    
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['credit' => $newBalance]);
    
    // Log transaction
    if ($amount > 0) {
        addTransaction([
            'clientid' => $clientId,
            'description' => $description ?: 'Credit added',
            'amountin' => $amount
        ]);
    } else {
        addTransaction([
            'clientid' => $clientId,
            'description' => $description ?: 'Credit applied',
            'amountout' => abs($amount)
        ]);
    }
    
    return true;
}
```

**Example:**
```php
// Add $50 credit to client
updateClientBalance(123, 50.00, 'Promotional credit');

// Deduct $10 from balance
updateClientBalance(123, -10.00, 'Monthly service fee');
```

### getClientBalance()

Gets a client's current credit balance.

```php
/**
 * Get client credit balance
 * 
 * @param int $clientId Client ID
 * @return float Credit balance
 */
function getClientBalance(int $clientId): float
{
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    return (float) ($client->credit ?? 0);
}
```

**Example:**
```php
$balance = getClientBalance(123);
echo "Available credit: $" . number_format($balance, 2);
```

## Transaction Reports

### getTransactionSummary()

Generates transaction summary statistics.

```php
/**
 * Get transaction summary
 * 
 * @param string $startDate Start date
 * @param string $endDate End date
 * @param string $gateway Filter by gateway
 * @return array Summary data
 */
function getTransactionSummary(
    string $startDate,
    string $endDate,
    string $gateway = ''
): array {
    $query = Capsule::table('tbltransections')
        ->selectRaw('
            COUNT(*) as total_count,
            SUM(amountin) as total_in,
            SUM(amountout) as total_out,
            AVG(amountin) as avg_transaction
        ')
        ->whereBetween('date', [$startDate, $endDate]);
    
    if ($gateway) {
        $query->where('gateway', $gateway);
    }
    
    $result = $query->first();
    
    return [
        'total_count' => $result->total_count ?? 0,
        'total_in' => $result->total_in ?? 0,
        'total_out' => $result->total_out ?? 0,
        'net' => ($result->total_in ?? 0) - ($result->total_out ?? 0),
        'avg_transaction' => $result->avg_transaction ?? 0
    ];
}
```

**Example:**
```php
$summary = getTransactionSummary('2024-01-01', '2024-12-31', 'stripe');

echo "Total transactions: {$summary['total_count']}\n";
echo "Total revenue: $" . number_format($summary['total_in'], 2) . "\n";
echo "Net income: $" . number_format($summary['net'], 2);
```

## Gateway Integration

### processPayment()

Processes payment through a gateway.

```php
/**
 * Process payment through gateway
 * 
 * @param int $invoiceId Invoice ID
 * @param string $gateway Gateway name
 * @param array $paymentData Payment data
 * @return array Result
 */
function processPayment(int $invoiceId, string $gateway, array $paymentData): array
{
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->first();
    
    if (!$invoice) {
        return ['success' => false, 'error' => 'Invoice not found'];
    }
    
    // Load gateway module
    $gatewayInterface = new WHMCS\Module\Gateway();
    $gatewayInterface->load($gateway);
    
    // Process payment
    $params = [
        'invoiceId' => $invoiceId,
        'description' => "Invoice #{$invoiceId}",
        'amount' => $invoice->total,
        'currency' => $invoice->currency,
        'clientId' => $invoice->userid,
        'paymentData' => $paymentData
    ];
    
    $result = $gatewayInterface->capture($params);
    
    if ($result['success']) {
        addTransaction([
            'clientid' => $invoice->userid,
            'description' => "Payment for Invoice #{$invoiceId}",
            'amountin' => $invoice->total,
            'gateway' => $gateway,
            'transid' => $result['transactionId'],
            'invoiceid' => $invoiceId,
            'fee' => $result['fee'] ?? 0
        ]);
    }
    
    return $result;
}
```

## Best Practices

1. **Always verify transactions** - Check gateway responses before recording
2. **Use transaction IDs** - Store external gateway transaction IDs
3. **Handle fees** - Record gateway fees for accurate accounting
4. **Log all operations** - Use logging functions for audit trails
5. **Implement idempotency** - Prevent duplicate transactions
6. **Use proper currency handling** - Store amounts in smallest unit

## Related Functions

- [whmcs-functions-invoices.md](whmcs-functions-invoices.md) - Invoice management
- [whmcs-functions-clients.md](whmcs-functions-clients.md) - Client operations
- [whmcs-schema-gateways.md](whmcs-schema-gateways.md) - Gateway configuration