# WHMCS Order Functions

Complete reference for order management functions in WHMCS.

## Overview

WHMCS provides comprehensive order management including order creation, processing, status management, and fulfillment.

## Order CRUD Operations

### createOrder()

Creates a new order.

```php
/**
 * Create a new order
 * 
 * @param array $data Order data
 * @param bool $sendEmail Send order notification
 * @param bool $autoSetup Auto-setup products
 * @param bool $autoDomain Auto-register domains
 * @return int Order ID
 */
function createOrder(
    array $data,
    bool $sendEmail = true,
    bool $autoSetup = true,
    bool $autoDomain = true
): int {
    return Capsule::table('tblorders')->insertGetId([
        'userid' => $data['clientid'],
        'ordernum' => generateOrderNumber(),
        'date' => date('Y-m-d H:i:s'),
        'nameservers' => $data['nameservers'] ?? '',
        'transfersecret' => $data['transfersecret'] ?? '',
        'regdate' => date('Y-m-d'),
        'completeddate' => null,
        'status' => 'Pending',
        'paymentmethod' => $data['paymentmethod'] ?? '',
        'ipaddress' => $data['ipaddress'] ?? $_SERVER['REMOTE_ADDR'] ?? '',
        'affiliateid' => $data['affiliateid'] ?? 0,
        'notes' => $data['notes'] ?? '',
        'paymentmethodname' => $data['paymentmethodname'] ?? '',
    ]);
}
```

**Example:**
```php
$orderId = createOrder([
    'clientid' => 123,
    'paymentmethod' => 'stripe',
    'affiliateid' => 5,
    'ipaddress' => $_SERVER['REMOTE_ADDR']
], true, true, true);
```

### getOrder()

Retrieves an order by ID.

```php
/**
 * Get order by ID
 * 
 * @param int $orderId Order ID
 * @return array|null Order data
 */
function getOrder(int $orderId): ?array
{
    $result = Capsule::table('tblorders')
        ->where('id', $orderId)
        ->first();
    
    return $result ? (array) $result : null;
}
```

**Example:**
```php
$order = getOrder(1001);

if ($order) {
    echo "Order #{$order['ordernum']} - {$order['status']}";
}
```

### getOrders()

Retrieves multiple orders with filtering.

```php
/**
 * Get orders with filters
 * 
 * @param array $filters Filter options
 * @param int $limit Number of records
 * @param int $offset Starting offset
 * @return array Orders
 */
function getOrders(array $filters = [], int $limit = 50, int $offset = 0): array
{
    $query = Capsule::table('tblorders')
        ->select('tblorders.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tblorders.userid')
        ->orderBy('tblorders.id', 'desc');
    
    if (!empty($filters['clientId'])) {
        $query->where('tblorders.userid', $filters['clientId']);
    }
    
    if (!empty($filters['status'])) {
        $query->where('tblorders.status', $filters['status']);
    }
    
    if (!empty($filters['paymentMethod'])) {
        $query->where('tblorders.paymentmethod', $filters['paymentMethod']);
    }
    
    if (!empty($filters['dateFrom'])) {
        $query->where('tblorders.date', '>=', $filters['dateFrom']);
    }
    
    if (!empty($filters['dateTo'])) {
        $query->where('tblorders.date', '<=', $filters['dateTo']);
    }
    
    return $query->limit($limit)->offset($offset)->get()->toArray();
}
```

**Example:**
```php
$pendingOrders = getOrders(['status' => 'Pending']);

foreach ($pendingOrders as $order) {
    echo "Order #{$order['ordernum']} from {$order['firstname']}\n";
}
```

### updateOrder()

Updates an existing order.

```php
/**
 * Update an order
 * 
 * @param int $orderId Order ID
 * @param array $data Updated data
 * @return bool Success status
 */
function updateOrder(int $orderId, array $data): bool
{
    return Capsule::table('tblorders')
        ->where('id', $orderId)
        ->update($data) > 0;
}
```

**Example:**
```php
updateOrder(1001, [
    'notes' => 'Customer requested expedited processing',
    'status' => 'Active'
]);
```

## Order Items

### addOrderItem()

Adds an item to an order.

```php
/**
 * Add item to order
 * 
 * @param int $orderId Order ID
 * @param array $data Item data
 * @return int Item ID
 */
function addOrderItem(int $orderId, array $data): int
{
    return Capsule::table('tblorderitems')->insertGetId([
        'orderid' => $orderId,
        'userid' => $data['clientid'] ?? 0,
        'type' => $data['type'] ?? 'Hosting',
        'relid' => $data['relid'] ?? 0,
        'productid' => $data['productid'] ?? 0,
        'domain' => $data['domain'] ?? '',
        'billingcycle' => $data['billingcycle'] ?? 'Monthly',
        'amount' => $data['amount'] ?? 0,
        'description' => $data['description'] ?? '',
        'qty' => $data['qty'] ?? 1,
    ]);
}
```

**Example:**
```php
addOrderItem(1001, [
    'clientid' => 123,
    'type' => 'Hosting',
    'productid' => 1,
    'domain' => 'example.com',
    'billingcycle' => 'Annual',
    'amount' => 119.88,
    'description' => 'Premium Hosting - Annual'
]);

addOrderItem(1001, [
    'clientid' => 123,
    'type' => 'DomainRegister',
    'productid' => 0,
    'domain' => 'example.com',
    'billingcycle' => 'DomainRegister',
    'amount' => 14.99,
    'description' => 'Domain Registration - example.com'
]);
```

### getOrderItems()

Retrieves all items for an order.

```php
/**
 * Get order items
 * 
 * @param int $orderId Order ID
 * @return array Items
 */
function getOrderItems(int $orderId): array
{
    return Capsule::table('tblorderitems')
        ->where('orderid', $orderId)
        ->get()
        ->toArray();
}
```

## Order Processing

### processOrder()

Processes an order through fulfillment.

```php
/**
 * Process an order
 * 
 * @param int $orderId Order ID
 * @return array Result
 */
function processOrder(int $orderId): array
{
    $order = getOrder($orderId);
    
    if (!$order) {
        return ['success' => false, 'error' => 'Order not found'];
    }
    
    $items = getOrderItems($orderId);
    $results = [];
    
    foreach ($items as $item) {
        $result = processOrderItem($item);
        $results[] = $result;
    }
    
    // Update order status
    $allSuccess = !in_array(false, array_column($results, 'success'));
    
    updateOrder($orderId, [
        'status' => $allSuccess ? 'Active' : 'Pending',
        'completeddate' => $allSuccess ? date('Y-m-d H:i:s') : null
    ]);
    
    return [
        'success' => $allSuccess,
        'order_id' => $orderId,
        'item_results' => $results
    ];
}

/**
 * Process individual order item
 * 
 * @param object $item Order item
 * @return array Result
 */
function processOrderItem(object $item): array
{
    try {
        switch ($item->type) {
            case 'Hosting':
                return provisionService($item);
            
            case 'ResellerHosting':
                return provisionResellerService($item);
            
            case 'VPS':
            case 'Server':
                return provisionServer($item);
            
            case 'DomainRegister':
                return registerDomain($item);
            
            case 'DomainTransfer':
                return transferDomain($item);
            
            case 'Domain':
                return activateDomain($item);
            
            case 'Addon':
                return provisionAddon($item);
            
            case 'Upgrade':
                return processUpgrade($item);
            
            default:
                return ['success' => false, 'error' => "Unknown item type: {$item->type}"];
        }
    } catch (Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}
```

**Example:**
```php
$result = processOrder(1001);

if ($result['success']) {
    echo "Order processed successfully";
} else {
    echo "Order processing failed: " . implode(', ', array_column($result['item_results'], 'error'));
}
```

## Order Status Management

```php
/**
 * Change order status
 * 
 * @param int $orderId Order ID
 * @param string $status New status
 * @param string $reason Reason for change
 * @return bool Success status
 */
function changeOrderStatus(int $orderId, string $status, string $reason = ''): bool
{
    $validStatuses = ['Pending', 'Active', 'Suspended', 'Cancelled', 'Fraud'];
    
    if (!in_array($status, $validStatuses)) {
        return false;
    }
    
    $oldOrder = getOrder($orderId);
    
    updateOrder($orderId, [
        'status' => $status,
        'completeddate' => $status === 'Active' ? date('Y-m-d H:i:s') : null
    ]);
    
    logAdminActivity(
        "Order #{$orderId} status changed from {$oldOrder['status']} to {$status}. Reason: {$reason}",
        'orders',
        'status_change',
        $_SESSION['adminid'] ?? 0
    );
    
    return true;
}
```

**Example:**
```php
// Mark as fraud
changeOrderStatus(1001, 'Fraud', 'Suspected fraudulent payment');

// Cancel order
changeOrderStatus(1001, 'Cancelled', 'Customer requested cancellation');
```

## Order Cancellation

### cancelOrder()

Cancels an order.

```php
/**
 * Cancel an order
 * 
 * @param int $orderId Order ID
 * @param bool $terminateServices Terminate associated services
 * @param bool $refund Refund payments
 * @return array Result
 */
function cancelOrder(
    int $orderId,
    bool $terminateServices = false,
    bool $refund = false
): array {
    $order = getOrder($orderId);
    
    if ($order['status'] === 'Cancelled') {
        return ['success' => false, 'error' => 'Order already cancelled'];
    }
    
    // Terminate services if requested
    if ($terminateServices) {
        $items = getOrderItems($orderId);
        foreach ($items as $item) {
            terminateService($item);
        }
    }
    
    // Process refund if requested
    if ($refund) {
        $invoices = getClientInvoices($order['userid'], 'Paid');
        // Refund logic...
    }
    
    changeOrderStatus($orderId, 'Cancelled', 'Order cancelled');
    
    return ['success' => true, 'order_id' => $orderId];
}
```

## Order Number Generation

```php
/**
 * Generate unique order number
 * 
 * @return string Order number
 */
function generateOrderNumber(): string
{
    $prefix = Config\Setting::getValue('OrderNumberPrefix') ?: 'ORD';
    $nextNumber = Config\Setting::getValue('OrderNumberCounter') ?: 1;
    
    // Increment counter
    Config\Setting::setValue('OrderNumberCounter', $nextNumber + 1);
    
    return $prefix . '-' . date('Y') . '-' . str_pad($nextNumber, 6, '0', STR_PAD_LEFT);
}
```

## Order Validation

```php
/**
 * Validate order before creation
 * 
 * @param array $data Order data
 * @return array Validation result
 */
function validateOrder(array $data): array
{
    $errors = [];
    
    // Validate client exists
    if (empty($data['clientid']) || !getClient($data['clientid'])) {
        $errors[] = 'Invalid client';
    }
    
    // Validate payment method
    if (empty($data['paymentmethod'])) {
        $errors[] = 'Payment method is required';
    }
    
    // Validate items
    if (empty($data['items']) || !is_array($data['items'])) {
        $errors[] = 'At least one item is required';
    }
    
    // Validate each item
    foreach ($data['items'] as $index => $item) {
        if (empty($item['productid']) && empty($item['domain'])) {
            $errors[] = "Item {$index}: Product or domain is required";
        }
    }
    
    return [
        'valid' => empty($errors),
        'errors' => $errors
    ];
}
```

## Order Search

```php
/**
 * Search orders
 * 
 * @param string $term Search term
 * @param array $fields Fields to search
 * @return array Matching orders
 */
function searchOrders(string $term, array $fields = []): array
{
    $defaultFields = ['ordernum', 'transfersecret'];
    $fields = $fields ?: $defaultFields;
    
    $query = Capsule::table('tblorders')
        ->select('tblorders.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tblorders.userid');
    
    foreach ($fields as $field) {
        $query->orWhere('tblorders.' . $field, 'like', '%' . $term . '%');
    }
    
    return $query->limit(50)->get()->toArray();
}
```

## Order Statistics

```php
/**
 * Get order statistics
 * 
 * @param string $startDate Start date
 * @param string $endDate End date
 * @return array Statistics
 */
function getOrderStatistics(string $startDate, string $endDate): array
{
    $stats = Capsule::table('tblorders')
        ->selectRaw("
            COUNT(*) as total_orders,
            SUM(amount) as total_revenue,
            COUNT(CASE WHEN status = 'Pending' THEN 1 END) as pending,
            COUNT(CASE WHEN status = 'Active' THEN 1 END) as active,
            COUNT(CASE WHEN status = 'Cancelled' THEN 1 END) as cancelled,
            COUNT(CASE WHEN status = 'Fraud' THEN 1 END) as fraud
        ")
        ->whereBetween('date', [$startDate, $endDate])
        ->first();
    
    return (array) $stats;
}
```

## Order Queue Processing

```php
/**
 * Process pending orders from queue
 * 
 * @param int $limit Maximum orders to process
 * @return array Processing results
 */
function processOrderQueue(int $limit = 100): array
{
    $pendingOrders = getOrders(['status' => 'Pending'], $limit);
    
    $processed = 0;
    $failed = 0;
    
    foreach ($pendingOrders as $order) {
        $result = processOrder($order['id']);
        
        if ($result['success']) {
            $processed++;
        } else {
            $failed++;
            
            // Log failure
            logActivity("Order #{$order['id']} processing failed", 0);
        }
    }
    
    return [
        'processed' => $processed,
        'failed' => $failed,
        'remaining' => count(getOrders(['status' => 'Pending']))
    ];
}
```

## Best Practices

1. **Always validate orders** - Check all data before processing
2. **Use transactions** - Wrap order creation and fulfillment in transactions
3. **Handle failures gracefully** - Implement proper error handling
4. **Log all operations** - Maintain audit trail for order changes
5. **Queue large operations** - Use background processing for heavy tasks
6. **Implement idempotency** - Prevent duplicate order processing

## Related Functions

- [whmcs-functions-products.md](whmcs-functions-products.md) - Product management
- [whmcs-functions-services.md](whmcs-functions-services.md) - Service provisioning
- [whmcs-functions-invoices.md](whmcs-functions-invoices.md) - Invoice generation
- [whmcs-schema-orders.md](whmcs-schema-orders.md) - Order database schema