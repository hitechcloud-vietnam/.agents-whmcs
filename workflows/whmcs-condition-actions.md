# WHMCS Conditional Actions Workflow

## Overview
This workflow implements conditional logic for automated actions based on client, service, and invoice states.

## Prerequisites
- WHMCS installation with custom hooks capability
- Admin access for configuration
- PHP development skills for advanced conditions

## Step-by-Step Process

### Step 1: Understand Conditional Logic Structure
```
IF condition_met THEN
    execute_action
ELSE IF alternative_condition THEN
    execute_alternative
ELSE
    execute_default
END IF
```

### Step 2: Create Conditional Action Hook
```php
<?php
// /includes/hooks/conditional_actions.php

/**
 * Conditional Action Handlers
 * Implements business logic with conditions
 */

// ============================================================================
// CLIENT-BASED CONDITIONS
// ============================================================================

add_hook('AfterServiceCreate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $clientId = $vars['userid'];

    // Get client information
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();

    // Get service details
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();

    $product = Capsule::table('tblproducts')
        ->where('id', $service->packageid)
        ->first();

    // CONDITION: New client ordering premium product
    if ($client->datecreated == date('Y-m-d') && $product->type == 'reserved') {
        // Action: Apply welcome bonus
        applyWelcomeCredit($clientId, 10);

        // Action: Assign to premium support queue
        assignPremiumSupport($serviceId);

        // Action: Send VIP welcome email
        sendEmail($clientId, 'vip_welcome', ['service' => $service]);
    }

    // CONDITION: Client has existing overdue invoices
    $overdueInvoices = Capsule::table('tblinvoices')
        ->where('userid', $clientId)
        ->where('status', 'Overdue')
        ->count();

    if ($overdueInvoices > 0) {
        // Action: Flag for manual review
        flagServiceForReview($serviceId, 'payment_hold');

        // Action: Send warning email
        sendEmail($clientId, 'payment_warning', [
            'overdue_count' => $overdueInvoices
        ]);
    }
});
```

### Step 3: Implement Service Status Conditions
```php
// Service suspension logic with conditions
add_hook('DailyCronJob', 1, function($vars) {
    // Get all active services with overdue invoices
    $services = Capsule::table('tblhosting')
        ->join('tblinvoices', 'tblhosting.userid', '=', 'tblinvoices.userid')
        ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
        ->where('tblhosting.domainstatus', 'Active')
        ->where('tblinvoices.status', 'Overdue')
        ->where('tblinvoices.duedate', '<', date('Y-m-d', strtotime('-14 days')))
        ->groupBy('tblhosting.id')
        ->get();

    foreach ($services as $service) {
        // CONDITION: Service is marked as protected
        if (isServiceProtected($service->id)) {
            continue; // Skip protected services
        }

        // CONDITION: Product type allows suspension
        if ($service->autosuspend != 'on') {
            continue; // Auto-suspend disabled for this product
        }

        // CONDITION: Client is VIP (no suspension)
        if (isClientVIP($service->userid)) {
            sendAdminAlert("VIP client has overdue invoice", $service);
            continue;
        }

        // Execute suspension action
        suspendService($service->id, 'Overdue Invoice');
    }
});
```

### Step 4: Create Invoice-Based Conditions
```php
// Invoice overdue handling with escalation
add_hook('InvoiceOverdue', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    $clientId = $vars['userid'];
    $daysOverdue = $vars['dues'];

    // Get client tier
    $client = getClient($clientId);
    $clientTier = $client['groupid'];

    // CONDITION: Calculate grace period based on client tier
    $gracePeriod = match ($clientTier) {
        1 => 3,   // Standard: 3 days
        2 => 7,   // Premium: 7 days
        3 => 14,  // VIP: 14 days
        default => 0
    };

    if ($daysOverdue >= $gracePeriod) {
        // CONDITION: Enterprise clients get warning first
        if ($clientTier == 4) { // Enterprise
            if ($daysOverdue == $gracePeriod) {
                sendEmail($clientId, 'enterprise_payment_warning', [
                    'days_overdue' => $daysOverdue
                ]);
                return;
            }
        }

        // Execute escalation actions
        escalateOverdueInvoice($invoiceId, $daysOverdue);
    }
});

function escalateOverdueInvoice($invoiceId, $daysOverdue)
{
    // Action: Suspend services
    suspendClientServices($invoiceId);

    // Action: Add late fee
    if ($daysOverdue >= 30) {
        addLateFee($invoiceId, 10.00);
    }

    // Action: Notify sales team
    if ($daysOverdue >= 14) {
        createInternalTicket("Overdue invoice escalation", $invoiceId);
    }

    // Action: Send final notice
    if ($daysOverdue >= 21) {
        sendEmail($clientId, 'final_payment_notice', [
            'days_overdue' => $daysOverdue,
            'action_required' => 'Payment or cancellation'
        ]);
    }

    // Action: Begin termination process
    if ($daysOverdue >= 45) {
        initiateTerminationProcess($invoiceId);
    }
}
```

### Step 5: Implement Product-Based Conditions
```php
// Product-specific provisioning logic
add_hook('AfterServiceCreate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $productId = $vars['packageid'];

    // Load product configuration
    $product = Capsule::table('tblproducts')
        ->where('id', $productId)
        ->first();

    $configOptions = Capsule::table('tblproductconfigoptions')
        ->join('tblproductconfiglinks', 'tblproductconfigoptions.id', '=', 'tblproductconfiglinks.gid')
        ->where('tblproductconfiglinks.relid', $productId)
        ->get();

    // CONDITION: Check if product has specific requirements
    $hasCustomRequirements = Capsule::table('tblproducts')
        ->where('id', $productId)
        ->whereNotNull('custom_required_fields')
        ->first();

    if ($hasCustomRequirements) {
        // Action: Validate requirements
        $isValid = validateProductRequirements($vars['userid'], $productId);

        if (!$isValid) {
            // Action: Flag for manual approval
            flagServiceForApproval($serviceId, 'requirements');
            sendAdminNotification("Service requires manual approval");
        }
    }

    // CONDITION: Check for add-on products
    $addons = getServiceAddons($serviceId);

    foreach ($addons as $addon) {
        // CONDITION: High-resource addons require capacity check
        if ($addon->resource_intensive) {
            $hasCapacity = checkServerCapacity($addon->server_group);

            if (!$hasCapacity) {
                // Action: Route to alternate server
                rerouteServiceToAlternateServer($serviceId);

                // Action: Notify admin
                logActivity("Service {$serviceId} routed to alternate server", $vars['userid']);
            }
        }
    }
});
```

### Step 6: Create Time-Based Conditions
```php
// Time-based service upgrades
add_hook('DailyCronJob', 1, function($vars) {
    $today = date('Y-m-d');

    // CONDITION: Services due for upgrade offer
    $services = Capsule::table('tblhosting')
        ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
        ->where('tblhosting.domainstatus', 'Active')
        ->whereRaw("DATEDIFF(tblhosting.regdate, ?) <= 90", [$today]) // First 90 days
        ->get();

    foreach ($services as $service) {
        // CONDITION: Only offer upgrade once
        $hasReceivedOffer = Capsule::table('mod_upgrade_offers')
            ->where('service_id', $service->id)
            ->where('offer_type', 'trial_upgrade')
            ->exists();

        if (!$hasReceivedOffer) {
            // Action: Send upgrade offer
            sendUpgradeOffer($service->id, 'trial_upgrade');

            // Record offer
            Capsule::table('mod_upgrade_offers')->insert([
                'service_id' => $service->id,
                'offer_type' => 'trial_upgrade',
                'offered_at' => date('Y-m-d H:i:s')
            ]);
        }
    }

    // CONDITION: Renewal window for annual services
    $renewalWindow = Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->where('billingcycle', 'Annual')
        ->whereRaw("DATEDIFF(nextduedate, ?) BETWEEN 0 AND 30", [$today])
        ->get();

    foreach ($renewalWindow as $service) {
        // CONDITION: Apply retention discount if eligible
        if (isRetentionEligible($service->userid)) {
            $discount = getRetentionDiscount($service->userid);

            if ($discount > 0) {
                // Action: Apply loyalty discount
                applyRenewalDiscount($service->id, $discount);
            }
        }
    }
});
```

### Step 7: Implement Multi-Condition Workflows
```php
// Complex multi-condition onboarding workflow
add_hook('ClientAdd', 1, function($vars) {
    $clientId = $vars['userid'];
    $client = getClient($clientId);

    // Evaluate multiple conditions
    $conditions = [
        'is_new' => $client['datecreated'] == date('Y-m-d'),
        'has_verified_email' => $client['email_verified'] == 1,
        'is_from_partner' => !empty($client['affiliate_id']),
        'is_international' => $client['country'] !== 'US',
        'wants_newsletter' => $client['newsletter'] == 1
    ];

    // CONDITION BLOCK: New domestic client
    if ($conditions['is_new'] && !$conditions['is_international']) {
        sendEmail($clientId, 'welcome_domestic', []);
        assignClientTag($clientId, 'new_domestic');
    }

    // CONDITION BLOCK: New international client
    if ($conditions['is_new'] && $conditions['is_international']) {
        sendEmail($clientId, 'welcome_international', []);
        setLocale($clientId, 'auto');
        assignClientTag($clientId, 'new_international');
    }

    // CONDITION BLOCK: Partner referred client
    if ($conditions['is_new'] && $conditions['is_from_partner']) {
        // Track affiliate conversion
        creditAffiliate($client['affiliate_id'], $clientId);

        // Send partner-specific welcome
        sendEmail($clientId, 'welcome_partner', [
            'partner_id' => $client['affiliate_id']
        ]);
    }

    // CONDITION BLOCK: Newsletter subscriber
    if ($conditions['wants_newsletter']) {
        subscribeToNewsletter($clientId);

        // Send welcome content
        sendEmail($clientId, 'newsletter_welcome', []);
    }

    // CONDITION BLOCK: Email not verified
    if ($conditions['is_new'] && !$conditions['has_verified_email']) {
        sendVerificationEmail($clientId);

        // Set up reminder
        scheduleTask('send_verification_reminder', $clientId, '+3 days');
    }
});
```

### Step 8: Create Condition Evaluation Helper
```php
<?php
// /includes/helpers/condition_evaluator.php

class ConditionEvaluator
{
    private $context;

    public function __construct($context = [])
    {
        $this->context = $context;
    }

    public function evaluate($condition, $value, $operator = '=')
    {
        $contextValue = $this->context[$condition] ?? null;

        return match ($operator) {
            '=' => $contextValue == $value,
            '!=' => $contextValue != $value,
            '>' => $contextValue > $value,
            '<' => $contextValue < $value,
            '>=' => $contextValue >= $value,
            '<=' => $contextValue <= $value,
            'IN' => in_array($contextValue, (array)$value),
            'NOT IN' => !in_array($contextValue, (array)$value),
            'CONTAINS' => strpos($contextValue, $value) !== false,
            'LIKE' => fnmatch($value, $contextValue),
            default => false
        };
    }

    public function and(array $conditions): bool
    {
        foreach ($conditions as $condition) {
            if (!$this->evaluate(
                $condition['field'],
                $condition['value'],
                $condition['operator'] ?? '='
            )) {
                return false;
            }
        }
        return true;
    }

    public function or(array $conditions): bool
    {
        foreach ($conditions as $condition) {
            if ($this->evaluate(
                $condition['field'],
                $condition['value'],
                $condition['operator'] ?? '='
            )) {
                return true;
            }
        }
        return false;
    }
}
```

### Step 9: Debugging Conditional Logic
```php
// Add logging to conditional actions
add_hook('AfterServiceCreate', 1, function($vars) {
    $startTime = microtime(true);
    $log = [];

    try {
        // Build context
        $context = buildConditionContext($vars);

        // Log context
        $log['context'] = $context;

        // Evaluate conditions
        $evaluator = new ConditionEvaluator($context);

        if ($evaluator->and([
            ['field' => 'is_new_client', 'operator' => '=', 'value' => true],
            ['field' => 'product_type', 'operator' => 'IN', 'value' => ['vps', 'server']]
        ])) {
            // Execute premium onboarding
            $log['action'] = 'premium_onboarding';
            executePremiumOnboarding($vars);
        }

        $log['success'] = true;
    } catch (Exception $e) {
        $log['success'] = false;
        $log['error'] = $e->getMessage();
    }

    $log['execution_time'] = microtime(true) - $startTime;

    // Store log
    Capsule::table('mod_condition_logs')->insert($log);
});
```

## Common Condition Patterns

| Condition Type | Example | Usage |
|----------------|---------|-------|
| Equality | `status == 'Active'` | Simple state check |
| Comparison | `amount > 100` | Threshold-based |
| Multiple | `type IN ['vps', 'server']` | Multi-value match |
| Date Range | `created >= last_30_days` | Time-based |
| Composite | `new AND premium` | Boolean logic |

## Best Practices

1. **Log all conditions** - Helps troubleshooting
2. **Use helper classes** - Reusable condition logic
3. **Test edge cases** - Empty values, null states
4. **Document business rules** - Maintain clarity
5. **Consider performance** - Avoid N+1 queries

## Related Workflows
- [WHMCS Automation Rules](./whmcs-automation-rules.md)
- [WHMCS Event-Driven Automation](./whmcs-event-driven-automation.md)
- [WHMCS Batch Operations](./whmcs-batch-operations.md)