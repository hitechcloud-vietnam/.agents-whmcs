# WHMCS Order Hooks

## Overview

Order hooks allow you to intercept and customize the order process in WHMCS.

## Available Order Hooks

### Order Created Hook

```php
<?php
// Triggered when a new order is placed
add_hook('OrderCreated', 1, function(array $vars) {
    $orderId = $vars['order_id'];
    $userId = $vars['user_id'];
    $products = $vars['products'];
    
    // Validate inventory
    validateProductInventory($products);
    
    // Apply order-specific pricing
    applyOrderPricing($orderId, $userId);
    
    // Sync to fulfillment system
    syncOrderToFulfillment($orderId);
    
    // Set up affiliate tracking
    processAffiliateTracking($orderId);
    
    // Send order confirmation
    sendOrderConfirmation($orderId);
    
    return ['success' => true, 'order_id' => $orderId];
});

function validateProductInventory(array $products): void
{
    foreach ($products as $product) {
        $stock = getProductStock($product['pid']);
        
        if ($stock !== null && $stock < $product['qty']) {
            throw new Exception("Insufficient stock for product {$product['pid']}");
        }
    }
}
```

### Order Accepted Hook

```php
<?php
// Triggered when an order is accepted/approved
add_hook('OrderAccepted', 1, function(array $vars) {
    $orderId = $vars['order_id'];
    $userId = $vars['user_id'];
    $products = $vars['products'];
    
    // Provision all products
    foreach ($products as $product) {
        provisionProduct($orderId, $product);
    }
    
    // Create invoices
    createOrderInvoices($orderId);
    
    // Setup client area access
    setupClientAccess($userId);
    
    // Send welcome emails
    sendWelcomeEmails($products);
    
    // Update analytics
    trackOrderMetrics($orderId);
    
    return ['success' => true];
});

function provisionProduct(int $orderId, array $product): void
{
    $module = new ServerModule($product['servertype']);
    
    // Create account
    $result = $module->createAccount($product['service_id']);
    
    if ($result['success']) {
        Capsule::table('tblhosting')
            ->where('id', $product['service_id'])
            ->update(['domainstatus' => 'Active']);
        
        // Log provisioning
        logProvisioning($product['service_id'], 'created', $result);
    }
}
```

### Order Rejected Hook

```php
<?php
// Triggered when an order is rejected
add_hook('OrderRejected', 1, function(array $vars) {
    $orderId = $vars['order_id'];
    $reason = $vars['reason'] ?? 'No reason provided';
    $userId = $vars['user_id'];
    
    // Log rejection
    logOrderRejection($orderId, $reason);
    
    // Release any inventory holds
    releaseInventoryHolds($orderId);
    
    // Reverse any applied credits
    reverseOrderCredits($orderId);
    
    // Notify customer
    notifyOrderRejection($userId, $orderId, $reason);
    
    // Alert fraud team if fraud-related
    if ($vars['fraud_check_failed'] ?? false) {
        alertFraudTeam($orderId);
    }
    
    return ['success' => true];
});
```

### Order Paid Hook

```php
<?php
// Triggered when order payment is received
add_hook('OrderPaid', 1, function(array $vars) {
    $orderId = $vars['order_id'];
    $paymentMethod = $vars['payment_method'];
    $amount = $vars['amount'];
    
    // Finalize provisioning
    finalizeOrderProvisioning($orderId);
    
    // Record payment
    recordOrderPayment($orderId, $paymentMethod, $amount);
    
    // Unlock any pending services
    unlockPendingServices($orderId);
    
    // Process affiliate commission
    processAffiliateCommissionForOrder($orderId, $amount);
    
    // Award loyalty points
    awardOrderLoyaltyPoints($vars['user_id'], $amount);
    
    return ['success' => true];
});

function finalizeOrderProvisioning(int $orderId): void
{
    $items = Capsule::table('tblorderproducts')
        ->where('orderid', $orderId)
        ->get();
    
    foreach ($items as $item) {
        // Send provisioning notification
        sendProvisioningEmail($item->id, $item->email);
    }
}
```

### Order Cancelled Hook

```php
<?php
// Triggered when an order is cancelled
add_hook('OrderCancelled', 1, function(array $vars) {
    $orderId = $vars['order_id'];
    $userId = $vars['user_id'];
    $reason = $vars['cancel_reason'] ?? '';
    $refundStatus = $vars['refund_status'];
    
    // Terminate services
    terminateOrderServices($orderId);
    
    // Handle refund if applicable
    if ($refundStatus === 'full' || $refundStatus === 'partial') {
        processOrderRefund($orderId, $refundStatus);
    }
    
    // Reverse affiliate commission
    reverseAffiliateCommission($orderId);
    
    // Log cancellation
    logOrderCancellation($orderId, $reason, $refundStatus);
    
    // Send cancellation confirmation
    sendCancellationConfirmation($userId, $orderId, $reason);
    
    return ['success' => true];
});

function terminateOrderServices(int $orderId): void
{
    $items = Capsule::table('tblorderproducts')
        ->where('orderid', $orderId)
        ->get();
    
    foreach ($items as $item) {
        if ($item->status === 'Active') {
            $module = new ServerModule($item->servertype);
            $module->terminateAccount($item->id, 'Order cancelled');
            
            Capsule::table('tblhosting')
                ->where('id', $item->id)
                ->update(['domainstatus' => 'Terminated']);
        }
    }
}
```

### Order Fraud Check Hook

```php
<?php
// Runs fraud checks on new orders
add_hook('OrderFraudCheck', 1, function(array $vars) {
    $orderId = $vars['order_id'];
    $userId = $vars['user_id'];
    $orderTotal = $vars['total'];
    
    $riskScore = 0;
    $riskFactors = [];
    
    // Check for high-risk countries
    if (in_array($vars['country'], getHighRiskCountries())) {
        $riskScore += 30;
        $riskFactors[] = 'high_risk_country';
    }
    
    // Check for free email domains
    if (isFreeEmailDomain($vars['email'])) {
        $riskScore += 20;
        $riskFactors[] = 'free_email_domain';
    }
    
    // Check order value against average
    $avgOrderValue = getAverageOrderValue($userId);
    if ($orderTotal > ($avgOrderValue * 3)) {
        $riskScore += 25;
        $riskFactors[] = 'unusual_order_value';
    }
    
    // Check for VPN/proxy
    if (isVpnOrProxy($_SERVER['REMOTE_ADDR'])) {
        $riskScore += 40;
        $riskFactors[] = 'vpn_detected';
    }
    
    // Check email reputation
    if (!checkEmailReputation($vars['email'])) {
        $riskScore += 15;
        $riskFactors[] = 'poor_email_reputation';
    }
    
    // Determine action based on score
    if ($riskScore >= 70) {
        return [
            'action' => 'reject',
            'risk_score' => $riskScore,
            'risk_factors' => $riskFactors,
            'reason' => 'High fraud risk detected',
        ];
    } elseif ($riskScore >= 40) {
        return [
            'action' => 'review',
            'risk_score' => $riskScore,
            'risk_factors' => $riskFactors,
        ];
    }
    
    return [
        'action' => 'accept',
        'risk_score' => $riskScore,
    ];
});
```

### Pre-Order Validation Hook

```php
<?php
// Runs before order is processed
add_hook('PreOrderCheck', 1, function(array $vars) {
    $userId = $vars['user_id'];
    $products = $vars['products'];
    $errors = [];
    $warnings = [];
    
    // Check for product conflicts
    foreach ($products as $product) {
        if (hasConflictingProduct($userId, $product['pid'])) {
            $errors[] = "You already have a conflicting product";
        }
    }
    
    // Check age restrictions
    foreach ($products as $product) {
        if (hasAgeRestriction($product['pid'])) {
            if (!$vars['age_verified']) {
                $errors[] = "Age verification required for this product";
            }
        }
    }
    
    // Check for promotional eligibility
    if (isset($vars['promo_code'])) {
        if (!isValidPromoCode($vars['promo_code'], $products)) {
            $errors[] = "Invalid or expired promotional code";
        }
    }
    
    if (!empty($errors)) {
        return [
            'abort' => true,
            'error_msg' => implode('. ', $errors),
        ];
    }
    
    if (!empty($warnings)) {
        return [
            'abort' => false,
            'warning_msg' => implode('. ', $warnings),
        ];
    }
    
    return ['abort' => false];
});
```

## Order Product Hooks

```php
<?php
// Triggered for each product in an order
add_hook('OrderProductAdd', 1, function(array $vars) {
    $orderId = $vars['order_id'];
    $productId = $vars['pid'];
    $serviceId = $vars['service_id'];
    
    // Add product-specific configuration
    configureProduct($serviceId, $vars['configoptions']);
    
    // Setup product addons
    setupProductAddons($serviceId, $vars['addons']);
    
    // Configure domain settings
    if (!empty($vars['domain'])) {
        configureDomain($vars['domain'], $serviceId);
    }
    
    return ['success' => true];
});
```

## Comprehensive Order Handler

```php
<?php
class OrderHookHandler {
    
    public function register(): void
    {
        add_hook('OrderCreated', 1, [$this, 'handleOrderCreated']);
        add_hook('OrderAccepted', 1, [$this, 'handleOrderAccepted']);
        add_hook('OrderRejected', 1, [$this, 'handleOrderRejected']);
        add_hook('OrderPaid', 1, [$this, 'handleOrderPaid']);
        add_hook('OrderCancelled', 1, [$this, 'handleOrderCancelled']);
        add_hook('OrderFraudCheck', 1, [$this, 'handleFraudCheck']);
        add_hook('PreOrderCheck', 1, [$this, 'handlePreOrder']);
    }
    
    public function handleOrderCreated(array $vars): array
    {
        $this->applyInitialPricing($vars['order_id']);
        $this->checkInventory($vars['products']);
        $this->syncToCrm($vars);
        
        return ['success' => true];
    }
    
    public function handleOrderAccepted(array $vars): array
    {
        $this->provisionProducts($vars['order_id']);
        $this->createInvoices($vars['order_id']);
        $this->notifyCustomer($vars['order_id']);
        
        return ['success' => true];
    }
    
    public function handleOrderRejected(array $vars): array
    {
        $this->releaseInventory($vars['order_id']);
        $this->notifyRejection($vars['order_id'], $vars['reason'] ?? '');
        
        return ['success' => true];
    }
    
    public function handleOrderPaid(array $vars): array
    {
        $this->finalizeProvisioning($vars['order_id']);
        $this->recordPayment($vars);
        $this->processAffiliates($vars['order_id']);
        
        return ['success' => true];
    }
    
    public function handleOrderCancelled(array $vars): array
    {
        $this->terminateServices($vars['order_id']);
        $this->handleRefund($vars);
        $this->logCancellation($vars);
        
        return ['success' => true];
    }
    
    public function handleFraudCheck(array $vars): array
    {
        $riskScore = $this->calculateRiskScore($vars);
        
        return [
            'action' => $riskScore >= 70 ? 'reject' : ($riskScore >= 40 ? 'review' : 'accept'),
            'risk_score' => $riskScore,
        ];
    }
    
    public function handlePreOrder(array $vars): array
    {
        $errors = $this->validateOrder($vars);
        
        if (!empty($errors)) {
            return ['abort' => true, 'error_msg' => implode('. ', $errors)];
        }
        
        return ['abort' => false];
    }
    
    private function applyInitialPricing(int $orderId): void
    {
        // Apply volume discounts, promotions, etc.
    }
    
    private function checkInventory(array $products): void
    {
        // Verify stock levels
    }
    
    private function syncToCrm(array $vars): void
    {
        // Sync order to external CRM
    }
    
    private function provisionProducts(int $orderId): void
    {
        // Provision all products in order
    }
    
    private function createInvoices(int $orderId): void
    {
        // Create/update invoices
    }
    
    private function notifyCustomer(int $orderId): void
    {
        // Send notifications
    }
    
    private function releaseInventory(int $orderId): void
    {
        // Release any inventory holds
    }
    
    private function notifyRejection(int $orderId, string $reason): void
    {
        // Send rejection notification
    }
    
    private function finalizeProvisioning(int $orderId): void
    {
        // Final provisioning steps
    }
    
    private function recordPayment(array $vars): void
    {
        // Record payment details
    }
    
    private function processAffiliates(int $orderId): void
    {
        // Process affiliate commissions
    }
    
    private function terminateServices(int $orderId): void
    {
        // Terminate all services
    }
    
    private function handleRefund(array $vars): void
    {
        // Handle refund if applicable
    }
    
    private function logCancellation(array $vars): void
    {
        // Log cancellation details
    }
    
    private function calculateRiskScore(array $vars): int
    {
        $score = 0;
        // Risk calculation logic
        return $score;
    }
    
    private function validateOrder(array $vars): array
    {
        $errors = [];
        // Validation logic
        return $errors;
    }
}

$handler = new OrderHookHandler();
$handler->register();
```

## Best Practices

1. **Always provision in OrderAccepted** - Never before payment confirmation
2. **Handle failures gracefully** - Implement rollback mechanisms
3. **Use transactions** - Ensure atomic operations
4. **Log extensively** - Track all order state changes
5. **Queue notifications** - Send emails asynchronously

## Related Documentation

- [WHMCS Invoice Hooks](/docs/whmcs-invoice-hooks.md)
- [WHMCS Service Hooks](/docs/whmcs-service-hooks.md)