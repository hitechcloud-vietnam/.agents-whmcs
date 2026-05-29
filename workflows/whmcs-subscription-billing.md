# WHMCS Subscription Billing Setup Workflow

## Description
Configure subscription/recurring billing for WHMCS products.

## Prerequisites
- Payment gateway with subscription support
- Recurring product pricing configured

## Steps

### Step 1: Configure Product for Subscription
```bash
# In WHMCS Admin:
# 1. Go to Configuration > System Settings > Products/Services
# 2. Create or edit product
# 3. Set Pricing > Billing Cycle: Recurring
# 4. Configure billing periods
```

### Step 2: Setup Gateway for Recurring
```php
<?php
// In gateway module

function clicodes_gateway_recurring($params)
{
    // Create subscription
    $subscription = [
        'customer' => $params['clientdetails']['email'],
        'price' => $params['amount'],
        'interval' => $params['billingcycle'], // monthly, yearly, etc.
        'currency' => $params['currency'],
    ];
    
    // Create subscription with payment provider
    $result = $api->createSubscription($subscription);
    
    if ($result['success']) {
        return [
            'subscriptionid' => $result['subscription_id'],
            'status' => 'active',
        ];
    }
    
    return ['error' => 'Failed to create subscription'];
}
```

### Step 3: Handle Subscription Updates
```php
<?php
// Webhook handler for subscription events

add_hook('SubscriptionUpdated', 1, function($vars) {
    $subscriptionId = $vars['subscription_id'];
    $newStatus = $vars['status'];
    
    if ($newStatus === 'past_due') {
        // Send warning email
        sendEmail('SubscriptionPastDue', $vars['user_id']);
    } elseif ($newStatus === 'canceled') {
        // Suspend service
        suspendAccount($vars['service_id']);
    }
});
```

### Step 4: Subscription Management
```php
<?php
// Client area subscription management

function subscription_management_page($params)
{
    $subscription = getSubscriptionDetails($params['subscription_id']);
    
    return [
        'templatefile' => 'clientarea_subscription',
        'vars' => [
            'subscription' => $subscription,
            'can_update_payment' => true,
            'can_cancel' => true,
        ],
    ];
}
```

## Subscription Workflow
1. Client orders product with recurring billing
2. Invoice generated automatically
3. Payment gateway charges on billing date
4. Service continues if payment successful
5. Service suspended if payment fails

## Tags
- subscription
- recurring-billing
- automation
- payment