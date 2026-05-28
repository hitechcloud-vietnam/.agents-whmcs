# WHMCS Subscription Workflow

## Purpose

Manage subscription-based billing, handle recurring payments, and automate subscription lifecycle events including upgrades, downgrades, and renewals.

## Prerequisites

- WHMCS with product/modules configured for recurring billing
- Payment gateway with subscription support
- Client products with billing cycles configured
- Cron job running for automated subscription processing

## Workflow Steps

### Step 1: Configure Subscription Products

Set up products for subscription billing:

```php
// Configure in WHMCS Admin > Setup > Products/Services
// Product Setup > Pricing Tab:
// - Billing Cycle: Recurring (Monthly/Quarterly/Annual)
// - Auto Setup: Enabled
// - Auto Terminate on Expiry: Configured

// Database configuration for products
Capsule::table('tblproducts')->where('id', $productId)->update([
    'paytype' => 'recurring',
    'billingcycle' => 'Monthly',
    'firstpaymentamount' => 0.00, // Setup fee
    'recurringamount' => 19.99,
    'cycles' => 0 // 0 = unlimited
]);
```

### Step 2: Create Subscription Management Module

Build custom subscription handling:

```php
// File: /includes/subscription_manager.php

class SubscriptionManager {
    
    public function createSubscription($clientId, $productId, $gateway) {
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();
        $product = Capsule::table('tblproducts')->where('id', $productId)->first();
        
        // Create the hosting account
        $hostingId = $this->createHostingAccount($clientId, $productId);
        
        // Create subscription with payment gateway
        $subscriptionId = $this->createGatewaySubscription($client, $product, $gateway);
        
        // Store subscription reference
        Capsule::table('mod_subscriptions')->insert([
            'client_id' => $clientId,
            'hosting_id' => $hostingId,
            'product_id' => $productId,
            'gateway' => $gateway,
            'gateway_subscription_id' => $subscriptionId,
            'status' => 'active',
            'next_billing_date' => date('Y-m-d', strtotime('+1 month')),
            'created_at' => Capsule::raw('NOW()')
        ]);
        
        return $subscriptionId;
    }
    
    private function createGatewaySubscription($client, $product, $gateway) {
        // Stripe example
        if ($gateway === 'stripe') {
            $stripe = new \Stripe\StripeClient($this->getApiKey($gateway));
            
            $subscription = $stripe->subscriptions->create([
                'customer' => $this->getOrCreateGatewayCustomer($client, $gateway),
                'items' => [['price' => $product->stripe_price_id]],
                'payment_behavior' => 'default_incomplete',
                'expand' => ['latest_invoice.payment_intent']
            ]);
            
            return $subscription->id;
        }
        
        return null;
    }
    
    public function handleSubscriptionUpdate($subscriptionId, $newPlan) {
        $subscription = Capsule::table('mod_subscriptions')
            ->where('gateway_subscription_id', $subscriptionId)
            ->first();
        
        // Update product association
        Capsule::table('tblhosting')
            ->where('id', $subscription->hosting_id)
            ->update(['packageid' => $newPlan['product_id']]);
        
        // Update gateway subscription
        $this->updateGatewaySubscription($subscriptionId, $newPlan);
        
        // Log the change
        Capsule::table('mod_subscription_changes')->insert([
            'subscription_id' => $subscription->id,
            'change_type' => 'plan_change',
            'old_plan' => $subscription->product_id,
            'new_plan' => $newPlan['product_id'],
            'effective_date' => date('Y-m-d'),
            'created_at' => Capsule::raw('NOW()')
        ]);
    }
    
    public function cancelSubscription($subscriptionId, $reason = 'customer_request') {
        $subscription = Capsule::table('mod_subscriptions')
            ->where('gateway_subscription_id', $subscriptionId)
            ->first();
        
        // Cancel with gateway
        $this->cancelGatewaySubscription($subscriptionId);
        
        // Update local records
        Capsule::table('mod_subscriptions')
            ->where('id', $subscription->id)
            ->update([
                'status' => 'cancelled',
                'cancelled_at' => Capsule::raw('NOW()'),
                'cancellation_reason' => $reason
            ]);
        
        // Schedule service termination
        Capsule::table('tblhosting')
            ->where('id', $subscription->hosting_id)
            ->update(['domainstatus' => 'Cancelled']);
        
        logActivity("Subscription {$subscriptionId} cancelled. Reason: {$reason}");
    }
}
```

### Step 3: Handle Subscription Webhooks

Process payment gateway subscription events:

```php
// File: /includes/hooks/subscription_webhooks.php

add_hook('WebhookCallback', 1, function($vars) {
    $type = $vars['type'] ?? '';
    $payload = $vars['payload'] ?? [];
    
    switch ($type) {
        case 'subscription.created':
            handleSubscriptionCreated($payload);
            break;
            
        case 'subscription.updated':
            handleSubscriptionUpdated($payload);
            break;
            
        case 'subscription.cancelled':
            handleSubscriptionCancelled($payload);
            break;
            
        case 'subscription.payment_succeeded':
            handleSubscriptionPaymentSucceeded($payload);
            break;
            
        case 'subscription.payment_failed':
            handleSubscriptionPaymentFailed($payload);
            break;
    }
});

function handleSubscriptionPaymentSucceeded($payload) {
    $subscriptionId = $payload['subscription_id'];
    
    $subscription = Capsule::table('mod_subscriptions')
        ->where('gateway_subscription_id', $subscriptionId)
        ->first();
    
    if ($subscription) {
        // Generate new invoice for next billing period
        $nextBillingDate = date('Y-m-d', strtotime('+1 month'));
        
        Capsule::table('mod_subscriptions')
            ->where('id', $subscription->id)
            ->update(['next_billing_date' => $nextBillingDate]);
        
        // Log successful payment
        logTransaction('Subscription Payment', $payload, 'Success');
        
        sendTemplatedEmail('SubscriptionPaymentSuccess', $subscription->client_id, [
            'amount' => $payload['amount_paid'],
            'next_billing_date' => $nextBillingDate
        ]);
    }
}

function handleSubscriptionPaymentFailed($payload) {
    $subscriptionId = $payload['subscription_id'];
    
    $subscription = Capsule::table('mod_subscriptions')
        ->where('gateway_subscription_id', $subscriptionId)
        ->first();
    
    if ($subscription) {
        // Update status
        Capsule::table('mod_subscriptions')
            ->where('id', $subscription->id)
            ->update(['status' => 'payment_failed']);
        
        // Notify client
        sendTemplatedEmail('SubscriptionPaymentFailed', $subscription->client_id, [
            'failure_reason' => $payload['failure_message'] ?? 'Payment failed',
            'retry_date' => date('Y-m-d', strtotime('+3 days'))
        ]);
        
        // Suspend service after grace period
        Capsule::table('tblhosting')
            ->where('id', $subscription->hosting_id)
            ->update(['domainstatus' => 'Suspended']);
        
        logActivity("Subscription payment failed for client #{$subscription->client_id}");
    }
}

function handleSubscriptionCancelled($payload) {
    $subscriptionId = $payload['subscription_id'];
    
    $subscription = Capsule::table('mod_subscriptions')
        ->where('gateway_subscription_id', $subscriptionId)
        ->first();
    
    if ($subscription) {
        Capsule::table('mod_subscriptions')
            ->where('id', $subscription->id)
            ->update([
                'status' => 'cancelled',
                'cancelled_at' => date('Y-m-d H:i:s')
            ]);
        
        // Terminate the service
        runModuleHook('Termination', $subscription->hosting_id);
        
        Capsule::table('tblhosting')
            ->where('id', $subscription->hosting_id)
            ->update(['domainstatus' => 'Terminated']);
    }
}
```

### Step 4: Implement Subscription Dunning

Handle failed payments with dunning process:

```php
// File: /includes/hooks/subscription_dunning.php

add_hook('DailyCronJob', 1, function($vars) {
    // Check for subscriptions with failed payments
    $failedSubscriptions = Capsule::table('mod_subscriptions')
        ->where('status', 'payment_failed')
        ->where('next_billing_date', '<=', date('Y-m-d'))
        ->get();
    
    foreach ($failedSubscriptions as $subscription) {
        // Retry payment
        $result = retrySubscriptionPayment($subscription->gateway_subscription_id);
        
        if ($result['success']) {
            Capsule::table('mod_subscriptions')
                ->where('id', $subscription->id)
                ->update(['status' => 'active', 'retry_count' => 0]);
            
            // Reactivate service
            Capsule::table('tblhosting')
                ->where('id', $subscription->hosting_id)
                ->update(['domainstatus' => 'Active']);
        } else {
            // Increment retry count
            $retryCount = ($subscription->retry_count ?? 0) + 1;
            
            Capsule::table('mod_subscriptions')
                ->where('id', $subscription->id)
                ->update(['retry_count' => $retryCount]);
            
            // After 3 retries, cancel subscription
            if ($retryCount >= 3) {
                $manager = new SubscriptionManager();
                $manager->cancelSubscription(
                    $subscription->gateway_subscription_id,
                    'Payment failed after ' . $retryCount . ' retries'
                );
                
                // Send final notice
                sendTemplatedEmail('SubscriptionCancelled', $subscription->client_id, [
                    'reason' => 'Payment failure'
                ]);
            }
        }
    }
    
    // Send reminder emails before billing date
    $reminders = Capsule::table('mod_subscriptions')
        ->where('status', 'active')
        ->where('next_billing_date', '=', date('Y-m-d', strtotime('+3 days')))
        ->get();
    
    foreach ($reminders as $subscription) {
        sendTemplatedEmail('SubscriptionRenewalReminder', $subscription->client_id, [
            'renewal_date' => $subscription->next_billing_date,
            'amount' => $subscription->recurring_amount
        ]);
    }
});
```

### Step 5: Subscription Analytics

Track subscription metrics:

```php
// File: /includes/hooks/subscription_analytics.php

function getSubscriptionMetrics() {
    $now = date('Y-m-d');
    $thirtyDaysAgo = date('Y-m-d', strtotime('-30 days'));
    
    return [
        'total_active' => Capsule::table('mod_subscriptions')
            ->where('status', 'active')
            ->count(),
        'total_cancelled' => Capsule::table('mod_subscriptions')
            ->where('status', 'cancelled')
            ->count(),
        'churn_rate' => calculateChurnRate($thirtyDaysAgo, $now),
        'monthly_recurring_revenue' => calculateMRR(),
        'new_this_month' => Capsule::table('mod_subscriptions')
            ->where('created_at', '>=', $thirtyDaysAgo)
            ->count(),
        'failed_payments' => Capsule::table('mod_subscriptions')
            ->where('status', 'payment_failed')
            ->count()
    ];
}

function calculateChurnRate($startDate, $endDate) {
    $startCount = Capsule::table('mod_subscriptions')
        ->where('created_at', '<', $startDate)
        ->whereIn('status', ['active', 'cancelled'])
        ->count();
    
    $cancelledCount = Capsule::table('mod_subscriptions')
        ->where('status', 'cancelled')
        ->whereBetween('cancelled_at', [$startDate, $endDate])
        ->count();
    
    return $startCount > 0 ? round(($cancelledCount / $startCount) * 100, 2) : 0;
}

function calculateMRR() {
    return Capsule::table('mod_subscriptions')
        ->where('status', 'active')
        ->sum('recurring_amount');
}
```

## Verification Checklist

- [ ] Products configured with recurring billing
- [ ] Subscription creation tested end-to-end
- [ ] Payment gateway subscription integration working
- [ ] Webhook handler processing events correctly
- [ ] Subscription updates (upgrade/downgrade) tested
- [ ] Subscription cancellation working properly
- [ ] Dunning process tested with failed payments
- [ ] Renewal reminders sent correctly
- [ ] Analytics data being collected accurately
- [ ] Subscription reports accessible in admin panel

## Related Skills and Documentation

- [WHMCS Recurring Billing](whmcs-recurring-billing-workflow.md)
- [WHMCS Subscription Cancellation](whmcs-subscription-cancellation-workflow.md)
- [WHMCS Payment Processing](whmcs-payment-processing-workflow.md)
- WHMCS Documentation: Recurring Billing
- WHMCS Documentation: Product Configuration

## Notes

- Test subscription lifecycle thoroughly before production
- Implement dunning carefully to minimize churn
- Monitor churn rate and identify improvement areas
- Consider trial periods for new subscriptions
- Keep subscription change history for audit
- Ensure PCI compliance with payment data handling