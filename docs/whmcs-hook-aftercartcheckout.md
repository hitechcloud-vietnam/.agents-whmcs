# WHMCS AfterCartCheckout Hook Reference

## Overview

The `AfterCartCheckout` hook fires after a cart checkout is completed in WHMCS. This hook triggers after payment processing and order creation, enabling post-purchase automation.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `orderid` | int | The created order ID |
| `userid` | int | Client ID |
| `invoiceid` | int | Generated invoice ID |
| `total` | float | Order total |
| `products` | array | Products ordered |
| `domains` | array | Domains ordered |
| `addons` | array | Addons ordered |
| `paymentmethod` | string | Payment gateway used |

## Example Implementation

```php
<?php
add_hook('AfterCartCheckout', 1, function(array $params) {
    // Log the checkout
    logActivity("Checkout completed: Order #{$params['orderid']} for user #{$params['userid']}");
    
    return $params;
});
```

## Order Fulfillment

```php
<?php
add_hook('AfterCartCheckout', 1, function(array $params) {
    $orderId = (int)$params['orderid'];
    
    // 1. Process each product type
    foreach ($params['products'] as $product) {
        if ($product['type'] === 'hosting') {
            // Additional hosting setup
            setupHostingExtras($product['id']);
        } elseif ($product['type'] === 'server') {
            // Server provisioning setup
            configureServerOptions($product['id']);
        }
    }
    
    // 2. Process domains
    foreach ($params['domains'] ?? [] as $domain) {
        configureDomainExtras($domain['id']);
    }
    
    // 3. Apply addons to services
    foreach ($params['addons'] ?? [] as $addon) {
        configureAddon($addon['id'], $addon['relid']);
    }
    
    return $params;
});
```

## Post-Purchase Notifications

```php
<?php
add_hook('AfterCartCheckout', 1, function(array $params) {
    $userId = (int)$params['userid'];
    $client = getClientsDetails($userId);
    
    // 1. Send order confirmation email
    sendTemplatedEmail('Order Confirmation', $client['email'], [
        'order_id' => $params['orderid'],
        'invoice_id' => $params['invoiceid'],
        'total' => $params['total']
    ]);
    
    // 2. Send SMS confirmation
    if ($client['phonenumber']) {
        sendSMS($client['phonenumber'], 
            "Order #{$params['orderid']} confirmed. Total: \${$params['total']}");
    }
    
    // 3. Post to Slack
    sendSlackMessage([
        'channel' => '#sales',
        'message' => "New order: #{$params['orderid']} for {$client['fullname']} - \${$params['total']}"
    ]);
    
    // 4. Notify sales team for high-value orders
    if ($params['total'] >= 500) {
        sendAdminNotification('sales', [
            'subject' => 'High Value Order',
            'message' => "Order #{$params['orderid']} - Total: \${$params['total']}"
        ]);
    }
    
    return $params;
});
```

## CRM and Marketing Integration

```php
<?php
add_hook('AfterCartCheckout', 1, function(array $params) {
    $userId = (int)$params['userid'];
    
    // 1. Update CRM
    updateCRMOrder($params['orderid'], [
        'user_id' => $userId,
        'total' => $params['total'],
        'products' => $params['products'],
        'timestamp' => date('c')
    ]);
    
    // 2. Track purchase in analytics
    trackPurchaseEvent([
        'order_id' => $params['orderid'],
        'revenue' => $params['total'],
        'items' => count($params['products']),
        'user_id' => $userId
    ]);
    
    // 3. Update marketing segments
    updateMarketingSegments($userId, [
        'last_purchase' => date('Y-m-d'),
        'lifetime_value' => getClientLifetimeValue($userId) + $params['total']
    ]);
    
    // 4. Start customer journey automation
    triggerAutomation('post_purchase_flow', $userId, [
        'order_id' => $params['orderid']
    ]);
    
    // 5. Update affiliate tracking
    creditAffiliateForOrder($userId, $params['orderid'], $params['total']);
    
    return $params;
});
```

## Retention and Loyalty

```php
<?php
add_hook('AfterCartCheckout', 1, function(array $params) {
    $userId = (int)$params['userid'];
    
    // 1. Award loyalty points
    $points = floor($params['total']); // 1 point per dollar
    addLoyaltyPoints($userId, $points);
    
    // 2. Check for tier upgrade
    checkTierUpgrade($userId);
    
    // 3. Create review request schedule
    scheduleReviewRequest($userId, $params['products']);
    
    // 4. Add to onboarding sequence
    addToOnboardingSequence($userId, $params['products']);
    
    // 5. Schedule follow-up
    scheduleFollowUp($userId, '+7 days', 'purchase_followup');
    
    return $params;
});
```

## Use Cases

- **Fulfillment**: Additional setup after order
- **Notifications**: Email, SMS, Slack alerts
- **CRM Integration**: Sync purchase data
- **Marketing**: Segment updates, automations
- **Loyalty**: Points, tier management

## Notes

- Runs after order creation, before provisioning
- Multiple hooks may fire (OrderPaid, InvoicePaid)
- Use for cross-sell recommendations
- Track for analytics and attribution

## Related Hooks

- `OrderPaid` - When order is paid
- `InvoicePaid` - When invoice is paid
- `CalculateCartTotals` - Cart calculation

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Order Processing](../whmcs-order-processing.md)