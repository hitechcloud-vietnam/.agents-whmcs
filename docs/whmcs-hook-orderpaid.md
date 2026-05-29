# WHMCS OrderPaid Hook Reference

## Overview

The `OrderPaid` hook fires when an order is marked as paid. This hook triggers after payment processing completes and the order transitions to paid status. It's essential for order fulfillment and automation workflows.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `orderid` | int | The unique order ID |
| `userid` | int | The client ID |
| `invoiceid` | int | The associated invoice ID |
| `total` | float | Order total amount |
| `status` | string | Order status (Active/Pending) |
| `paymentmethod` | string | Payment gateway used |
| `transid` | string | Transaction ID |
| `items` | array | Order line items |
| `fqdn` | string | Primary domain (if domain order) |
| `model` | object | The Order model instance |

## Example Implementation

```php
<?php
add_hook('OrderPaid', 1, function(array $params) {
    // Log the order payment
    logActivity("Order #{$params['orderid']} paid: {$params['total']}");
    
    // Get order details
    $order = $params['model'];
    $orderItems = $order->getLineItems();
    
    foreach ($orderItems as $item) {
        processOrderItem($item);
    }
    
    return $params;
});
```

## Fulfillment Workflow

```php
<?php
add_hook('OrderPaid', 1, function(array $params) {
    $orderId = (int)$params['orderid'];
    
    // Get all order items
    $result = full_query("
        SELECT * FROM tblorderitems WHERE orderid = $orderId
    ");
    
    while ($item = mysql_fetch_array($result)) {
        // Process based on product type
        switch ($item['type']) {
            case 'Hosting':
                // Service is typically provisioned automatically
                logActivity("Hosting service {$item['id']} for order {$orderId}");
                
                // Add custom configuration if needed
                if ($item['pid'] == 5) { // Specific product
                    applyCustomServiceConfig($item['id']);
                }
                break;
                
            case 'Domain':
                // Register/transfer domain
                registerDomainIfNotExists($item['id']);
                break;
                
            case 'Addon':
                // Apply addon to existing service
                applyAddonToService($item['id'], $item['relid']);
                break;
                
            case 'Upgrade':
                // Process upgrade request
                processUpgrade($item['id']);
                break;
        }
    }
    
    return $params;
});
```

## Automated Actions

```php
<?php
add_hook('OrderPaid', 1, function(array $params) {
    $orderId = (int)$params['orderid'];
    $userId = (int)$params['userid'];
    
    // 1. Grant loyalty points
    $points = floor($params['total']); // 1 point per dollar
    addLoyaltyPoints($userId, $points);
    
    // 2. Send welcome kit for first order
    $previousOrders = countOrdersForClient($userId);
    if ($previousOrders == 1) {
        sendWelcomeKit($userId);
    }
    
    // 3. Add to affiliate tracking
    creditAffiliate($userId, $params['total']);
    
    // 4. Create onboarding ticket
    createOnboardingTicket($orderId, $userId);
    
    // 5. Schedule follow-up
    scheduleFollowUp($orderId, '+7 days');
    
    // 6. Add to marketing segments
    updateMarketingSegments($userId, 'has_purchased');
    
    return $params;
});
```

## Third-Party Integrations

```php
<?php
add_hook('OrderPaid', 1, function(array $params) {
    // 1. Send to fulfillment service (e.g., Zapier)
    sendWebhook('order.paid', [
        'order_id' => $params['orderid'],
        'user_id' => $params['userid'],
        'total' => $params['total'],
        'items' => $params['items'],
        'timestamp' => date('c')
    ]);
    
    // 2. Update CRM
    updateCRMOrderStatus($params['orderid'], 'paid');
    
    // 3. Sync to inventory management
    syncOrderToInventory($params['orderid']);
    
    // 4. Post to analytics
    trackOrderEvent('purchase', [
        'order_id' => $params['orderid'],
        'revenue' => $params['total'],
        'items' => count($params['items'])
    ]);
    
    // 5. Add to email marketing list
    addToEmailList($params['userid'], 'customers');
    
    return $params;
});
```

## Validation and Control

```php
<?php
add_hook('OrderPaid', 1, function(array $params) {
    // Check for fraud indicators
    $fraudScore = calculateFraudScore($params['userid'], $params['orderid']);
    
    if ($fraudScore > 80) {
        // Flag for manual review instead of fulfilling
        update_query('tblorders', [
            'status' => 'Pending'
        ], ['id' => $params['orderid']]);
        
        sendAdminNotification('security', [
            'subject' => "High Fraud Risk Order",
            'message' => "Order #{$params['orderid']} flagged for manual review"
        ]);
        
        return [
            'success' => false,
            'errorMessage' => 'Order flagged for manual review'
        ];
    }
    
    return $params;
});
```

## Use Cases

- **Service Provisioning**: Activate hosting services
- **Domain Registration**: Register or transfer domains
- **Integrations**: Sync with CRM, accounting, analytics
- **Marketing**: Add to segments, trigger campaigns
- **Loyalty Programs**: Award points or rewards

## Notes

- Fires when order payment is confirmed
- Multiple hooks may fire for the same order (OrderPaid, InvoicePaid)
- Can return false to prevent automatic fulfillment
- Consider combining with `AfterCartCheckout` for cart-based orders

## Related Hooks

- `InvoicePaid` - When invoice is paid
- `AfterCartCheckout` - After cart checkout
- `AfterProductCreate` - After service creation

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Order Management](../whmcs-order-processing.md)