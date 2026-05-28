# WHMCS Subscription Management Module

Advanced subscription management module for WHMCS with configurable billing cycles, proration support, and automated renewal handling.

## Features

- Multiple billing cycles (monthly, quarterly, semi-annually, annually, biennially, triennially)
- Automatic proration calculation when changing plans
- Subscription pause/resume functionality
- Scheduled renewal processing
- Custom subscription plans with flexible pricing
- Event logging and history tracking
- Grace period management
- Renewal notice automation

## Installation

1. Copy the module files to your WHMCS installation:
   ```
   /path/to/whmcs/modules/servers/subscriptionmanagement/
   ```

2. Activate the module through WHMCS Admin:
   - Go to **Setup > Addon Modules**
   - Find "Subscription Management"
   - Click **Activate**
   - Configure module settings

3. Configure module settings:
   - **Default Billing Cycle**: Set the default cycle for new subscriptions
   - **Enable Proration**: Toggle proration calculation
   - **Proration Method**: Choose how proration differences are handled
   - **Auto Renew**: Enable automatic renewal attempts
   - **Renewal Notice Days**: Configure reminder timing
   - **Grace Period Days**: Set expiration grace period

## Usage

### Creating a Subscription

```php
// Create a new subscription
$result = subscriptionmanagement_CreateSubscription(
    $userId,           // Client user ID
    $relId,            // Service/Product ID
    $planKey,          // Plan identifier
    array(
        'billing_cycle' => 'monthly',
        'currency' => 'USD',
        'custom_fields' => array('key' => 'value'),
    )
);

if ($result['success']) {
    $subscriptionId = $result['subscription_id'];
    $nextBillingDate = $result['next_billing_date'];
}
```

### Managing Subscriptions

```php
// Get subscription details
$subscription = subscriptionmanagement_GetSubscription($subscriptionId);

// Change plan
$result = subscriptionmanagement_ChangePlan($subscriptionId, $newPlanId, 'quarterly');

// Pause subscription for 30 days
$result = subscriptionmanagement_PauseSubscription($subscriptionId, 30);

// Resume paused subscription
$result = subscriptionmanagement_ResumeSubscription($subscriptionId);

// Cancel subscription
$result = subscriptionmanagement_CancelSubscription($subscriptionId, false, 'User requested');
```

### Working with Plans

```php
// Create a new plan
$result = subscriptionmanagement_CreatePlan(array(
    'plan_name' => 'Premium Plan',
    'description' => 'Our premium offering',
    'billing_cycles' => array('monthly', 'quarterly', 'annually'),
    'pricing' => array(
        'monthly' => 29.99,
        'quarterly' => 79.99,
        'annually' => 249.99,
    ),
    'features' => array('Feature 1', 'Feature 2', 'Feature 3'),
));

// Get all active plans
$plans = subscriptionmanagement_GetPlans(true);

// Update plan
subscriptionmanagement_UpdatePlan($planKey, array(
    'pricing' => array('monthly' => 34.99),
));

// Delete/deactivate plan
subscriptionmanagement_DeletePlan($planKey);
```

### Getting Information

```php
// Get user's subscriptions
$subscriptions = subscriptionmanagement_GetSubscriptionsByUser($userId, array(
    'status' => 'active',
));

// Get due renewals
$dueRenewals = subscriptionmanagement_GetDueRenewals(3);

// Get statistics
$stats = subscriptionmanagement_GetStatistics($userId);
// Returns: total, active, paused, cancelled, expired, monthly_recurring_revenue

// Get subscription events
$events = subscriptionmanagement_GetEvents($subscriptionId);
```

## Cron Job Setup

Add to WHMCS cron scheduler for automatic renewals:

```bash
# In your WHMCS crons/crons/ directory, create a cron for subscription processing
# Schedule: Daily
```

The module will automatically:
- Process renewals on their due dates
- Handle expired subscriptions
- Process grace period expirations
- Generate renewal invoices

## Database Tables

The module creates three tables:

- `mod_subscription_management` - Main subscription data
- `mod_subscription_plans` - Subscription plan definitions
- `mod_subscription_events` - Event history and audit log

## API Functions

| Function | Description |
|----------|-------------|
| `subscriptionmanagement_CreateSubscription()` | Create new subscription |
| `subscriptionmanagement_GetSubscription()` | Get subscription details |
| `subscriptionmanagement_ChangePlan()` | Change subscription plan |
| `subscriptionmanagement_PauseSubscription()` | Pause subscription |
| `subscriptionmanagement_ResumeSubscription()` | Resume subscription |
| `subscriptionmanagement_CancelSubscription()` | Cancel subscription |
| `subscriptionmanagement_ProcessRenewal()` | Process subscription renewal |
| `subscriptionmanagement_CreatePlan()` | Create subscription plan |
| `subscriptionmanagement_GetPlans()` | Get all plans |
| `subscriptionmanagement_UpdatePlan()` | Update plan |
| `subscriptionmanagement_DeletePlan()` | Delete/deactivate plan |
| `subscriptionmanagement_GetStatistics()` | Get statistics |
| `subscriptionmanagement_GetEvents()` | Get subscription events |

## Version History

- **1.0.0** - Initial release
  - Basic subscription management
  - Multiple billing cycles
  - Proration support
  - Plan management
  - Event logging
