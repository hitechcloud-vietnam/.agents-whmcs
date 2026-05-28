# WHMCS Client Onboarding Workflow

## Purpose

Comprehensive guide to automating and optimizing the client onboarding process in WHMCS, from signup to first service delivery.

## Prerequisites

- WHMCS installation with cart/order system
- Payment gateway configured
- Products/services set up
- Email templates configured
- Optional: Provisioning modules ready

## Workflow Steps

### Step 1: Onboarding Workflow Design

Design the complete onboarding journey:

```php
// Onboarding stages configuration

return [
    'stages' => [
        'signup' => [
            'name' => 'Account Creation',
            'steps' => [
                'personal_info' => [
                    'required' => true,
                    'fields' => ['firstname', 'lastname', 'email', 'phonenumber'],
                    'validation' => 'standard',
                ],
                'company_info' => [
                    'required' => false,
                    'fields' => ['companyname', 'tax_id'],
                ],
                'address' => [
                    'required' => true,
                    'fields' => ['address1', 'city', 'state', 'postcode', 'country'],
                ],
            ],
            'auto_actions' => [
                'create_account' => true,
                'send_welcome_email' => true,
            ],
        ],
        'order' => [
            'name' => 'Product Selection',
            'steps' => [
                'select_product' => true,
                'configure_options' => true,
                'select_cycle' => true,
                'add_domains' => false,
            ],
            'auto_actions' => [
                'validate_availability' => true,
                'calculate_pricing' => true,
            ],
        ],
        'payment' => [
            'name' => 'Payment',
            'steps' => [
                'select_payment_method' => true,
                'enter_payment_info' => true,
                'apply_coupon' => false,
            ],
            'auto_actions' => [
                'create_invoice' => true,
                'process_payment' => true,
            ],
        ],
        'provisioning' => [
            'name' => 'Service Setup',
            'steps' => [
                'provision_service' => true,
                'send_credentials' => true,
                'setup_automations' => true,
            ],
        ],
        'completion' => [
            'name' => 'Welcome',
            'steps' => [
                'send_welcome_package' => true,
                'setup_support_access' => true,
                'invite_feedback' => false,
            ],
        ],
    ],
    
    'timeouts' => [
        'cart_abandonment' => 72, // hours
        'payment_pending' => 48,
        'verification_reminder' => 24,
    ],
];
```

### Step 2: Automated Onboarding Hooks

Implement onboarding automation:

```php
// modules/addons/onboarding_automation/onboarding.php

/**
 * Hook: After new client registration
 */
add_hook('ClientAdd', 1, function($vars) {
    $clientId = $vars['user_id'];
    
    // 1. Create onboarding record
    Capsule::table('mod_onboarding')->insert([
        'client_id' => $clientId,
        'stage' => 'signup_completed',
        'started_at' => date('Y-m-d H:i:s'),
        'metadata' => json_encode([
            'email_verified' => false,
            'profile_complete' => false,
            'first_order_placed' => false,
        ]),
    ]);
    
    // 2. Send verification email
    send_email('client_email_verification', [
        'id' => $clientId,
    ]);
    
    // 3. Set up welcome discount
    Capsule::table('mod_client_discounts')->insert([
        'client_id' => $clientId,
        'discount_type' => 'percentage',
        'discount_value' => 10,
        'max_uses' => 1,
        'expires_at' => date('Y-m-d H:i:s', strtotime('+30 days')),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // 4. Assign to welcome segment
    assignToMarketingSegment($clientId, 'welcome');
});

/**
 * Hook: After successful order
 */
add_hook('OrderPaid', 1, function($vars) {
    $orderId = $vars['order_id'];
    $clientId = $vars['user_id'];
    
    // 1. Update onboarding stage
    Capsule::table('mod_onboarding')
        ->where('client_id', $clientId)
        ->update([
            'stage' => 'first_order_completed',
            'first_order_at' => date('Y-m-d H:i:s'),
        ]);
    
    // 2. Get ordered services
    $services = Capsule::table('tblhosting')
        ->where('orderid', $orderId)
        ->get();
    
    // 3. Schedule provisioning notifications
    foreach ($services as $service) {
        scheduleProvisioningFollowup($clientId, $service->id);
    }
    
    // 4. Send order confirmation
    send_email('order_confirmation', [
        'id' => $orderId,
    ]);
    
    // 5. Add to onboarding checklist
    createOnboardingChecklist($clientId, $services);
});

/**
 * Hook: After service provisioning
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $serviceId = $vars['service_id'];
    $clientId = $vars['userid'];
    
    // 1. Update onboarding checklist
    Capsule::table('mod_onboarding_checklist')
        ->where('client_id', $clientId)
        ->where('service_id', $serviceId)
        ->update([
            'provisioning_complete' => true,
            'provisioned_at' => date('Y-m-d H:i:s'),
        ]);
    
    // 2. Send welcome email with credentials
    send_email('service_welcome_credentials', [
        'id' => $serviceId,
    ]);
    
    // 3. Schedule follow-up emails
    scheduleWelcomeFollowups($clientId, $serviceId);
    
    // 4. Check if onboarding complete
    checkOnboardingCompletion($clientId);
});

/**
 * Schedule onboarding follow-up emails
 */
function scheduleWelcomeFollowups(int $clientId, int $serviceId): void
{
    $followups = [
        ['days' => 1, 'template' => 'welcome_day1'],
        ['days' => 3, 'template' => 'getting_started_guide'],
        ['days' => 7, 'template' => 'welcome_week1_survey'],
        ['days' => 14, 'template' => 'check_in'],
    ];
    
    foreach ($followups as $followup) {
        Capsule::table('mod_onboarding_emails')->insert([
            'client_id' => $clientId,
            'service_id' => $serviceId,
            'template' => $followup['template'],
            'scheduled_date' => date('Y-m-d H:i:s', strtotime('+' . $followup['days'] . ' days')),
            'status' => 'pending',
        ]);
    }
}
```

### Step 3: Onboarding Checklist System

Create interactive onboarding checklist:

```php
// modules/addons/onboarding_checklist/checklist.php

/**
 * Get client onboarding checklist
 */
function getClientOnboardingChecklist(int $clientId): array
{
    $checklist = Capsule::table('mod_onboarding_checklist')
        ->where('client_id', $clientId)
        ->where('completed', 0)
        ->get();
    
    $items = [];
    
    foreach ($checklist as $item) {
        $items[] = [
            'id' => $item->id,
            'title' => $item->title,
            'description' => $item->description,
            'action_url' => $item->action_url,
            'action_text' => $item->action_text,
            'priority' => $item->priority,
            'completed' => (bool)$item->completed,
        ];
    }
    
    // Sort by priority
    usort($items, fn($a, $b) => $a['priority'] <=> $b['priority']);
    
    return $items;
}

/**
 * Create onboarding checklist for new client
 */
function createOnboardingChecklist(int $clientId, $services): void
{
    $defaultItems = [
        [
            'title' => 'Verify Your Email',
            'description' => 'Click the verification link in your email',
            'action_url' => 'verifyemail.php',
            'action_text' => 'Verify Email',
            'priority' => 1,
        ],
        [
            'title' => 'Complete Your Profile',
            'description' => 'Add your company information and preferences',
            'action_url' => 'clientarea.php?action=account',
            'action_text' => 'Edit Profile',
            'priority' => 2,
        ],
        [
            'title' => 'Set Up Two-Factor Authentication',
            'description' => 'Secure your account with 2FA',
            'action_url' => 'clientarea.php?action=security',
            'action_text' => 'Enable 2FA',
            'priority' => 3,
        ],
        [
            'title' => 'Add Payment Method',
            'description' => 'Add a credit card or bank account for easy payments',
            'action_url' => 'clientarea.php?action=payment_methods',
            'action_text' => 'Add Payment Method',
            'priority' => 4,
        ],
    ];
    
    foreach ($defaultItems as $item) {
        Capsule::table('mod_onboarding_checklist')->insert([
            'client_id' => $clientId,
            'title' => $item['title'],
            'description' => $item['description'],
            'action_url' => $item['action_url'],
            'action_text' => $item['action_text'],
            'priority' => $item['priority'],
            'completed' => 0,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    // Add service-specific items
    foreach ($services as $service) {
        Capsule::table('mod_onboarding_checklist')->insert([
            'client_id' => $clientId,
            'service_id' => $service->id,
            'title' => 'Set Up Your Service',
            'description' => "Configure your {$service->domain} service",
            'action_url' => "clientarea.php?action=productdetails&id={$service->id}",
            'action_text' => 'Configure Service',
            'priority' => 5,
            'completed' => 0,
        ]);
    }
}

/**
 * Mark checklist item complete
 */
function completeChecklistItem(int $itemId, int $clientId): bool
{
    $item = Capsule::table('mod_onboarding_checklist')
        ->where('id', $itemId)
        ->where('client_id', $clientId)
        ->first();
    
    if (!$item) {
        return false;
    }
    
    Capsule::table('mod_onboarding_checklist')
        ->where('id', $itemId)
        ->update([
            'completed' => 1,
            'completed_at' => date('Y-m-d H:i:s'),
        ]);
    
    // Trigger any completion actions
    triggerChecklistItemActions($item);
    
    // Check overall completion
    checkOnboardingCompletion($clientId);
    
    return true;
}
```

### Step 4: Progress Tracking Dashboard

Create onboarding progress tracking:

```php
// modules/addons/onboarding_dashboard/dashboard.php

/**
 * Get onboarding progress for client
 */
function getOnboardingProgress(int $clientId): array
{
    $onboarding = Capsule::table('mod_onboarding')
        ->where('client_id', $clientId)
        ->first();
    
    if (!$onboarding) {
        return ['started' => false];
    }
    
    $totalItems = Capsule::table('mod_onboarding_checklist')
        ->where('client_id', $clientId)
        ->count();
    
    $completedItems = Capsule::table('mod_onboarding_checklist')
        ->where('client_id', $clientId)
        ->where('completed', 1)
        ->count();
    
    $progress = $totalItems > 0 ? round(($completedItems / $totalItems) * 100) : 0;
    
    return [
        'started' => true,
        'stage' => $onboarding->stage,
        'started_at' => $onboarding->started_at,
        'progress_percentage' => $progress,
        'total_items' => $totalItems,
        'completed_items' => $completedItems,
        'pending_items' => $totalItems - $completedItems,
    ];
}

/**
 * Render onboarding progress widget
 */
function renderOnboardingProgressWidget(int $clientId): string
{
    $progress = getOnboardingProgress($clientId);
    $checklist = getClientOnboardingChecklist($clientId);
    
    if (!$progress['started']) {
        return '';
    }
    
    $html = <<<HTML
    <div class="onboarding-widget" style="background: #f8fafc; border-radius: 8px; padding: 20px; margin: 20px 0;">
        <h3 style="margin: 0 0 15px;">Setup Progress</h3>
        
        <div class="progress-bar" style="background: #e2e8f0; height: 10px; border-radius: 5px; overflow: hidden;">
            <div class="progress-fill" style="background: #2563EB; height: 100%; width: {$progress['progress_percentage']}%;"></div>
        </div>
        
        <p style="margin: 10px 0; color: #64748b;">
            {$progress['completed_items']} of {$progress['total_items']} tasks completed ({$progress['progress_percentage']}%)
        </p>
        
        <div class="checklist">
    HTML;
    
    foreach (array_slice($checklist, 0, 3) as $item) {
        $statusIcon = $item['completed'] ? '✓' : '○';
        $statusClass = $item['completed'] ? 'completed' : 'pending';
        
        $html .= <<<HTML
            <div class="checklist-item {$statusClass}" style="padding: 10px 0; border-bottom: 1px solid #e2e8f0;">
                <span style="margin-right: 10px;">{$statusIcon}</span>
                <a href="{$item['action_url']}">{$item['title']}</a>
            </div>
        HTML;
    }
    
    $html .= <<<HTML
        </div>
        
        <a href="onboarding.php" class="btn" style="display: inline-block; margin-top: 15px; background: #2563EB; color: white; padding: 10px 20px; text-decoration: none; border-radius: 5px;">
            View All Tasks
        </a>
    </div>
    HTML;
    
    return $html;
}
```

### Step 5: Abandoned Onboarding Recovery

Implement recovery for abandoned signups:

```php
// modules/addons/onboarding_recovery/recovery.php

/**
 * Check for abandoned onboarding
 */
add_hook('DailyCronJob', 1, function($vars) {
    // Check signup abandonment (24 hours)
    $abandonedSignups = Capsule::table('mod_onboarding')
        ->where('stage', 'signup_completed')
        ->where('first_order_at', null)
        ->where('started_at', '<', date('Y-m-d H:i:s', strtotime('-24 hours')))
        ->where('reminder_sent', 0)
        ->get();
    
    foreach ($abandonedSignups as $onboarding) {
        // Send reminder
        send_abandoned_signup_email($onboarding->client_id);
        
        // Mark as reminded
        Capsule::table('mod_onboarding')
            ->where('id', $onboarding->id)
            ->update(['reminder_sent' => 1, 'reminder_at' => date('Y-m-d H:i:s')]);
    }
    
    // Check cart abandonment
    checkCartAbandonment();
    
    // Check payment pending
    checkPaymentPending();
});

/**
 * Send abandoned signup recovery email
 */
function send_abandoned_signup_email(int $clientId): void
{
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    $discount = Capsule::table('mod_client_discounts')
        ->where('client_id', $clientId)
        ->where('used', 0)
        ->first();
    
    send_email('onboarding_abandoned_recovery', [
        'id' => $clientId,
    ], [
        'first_name' => $client->firstname,
        'discount_code' => $discount->discount_code ?? null,
        'discount_value' => $discount->discount_value ?? 0,
    ]);
}

/**
 * Track onboarding funnel analytics
 */
function trackOnboardingAnalytics(): array
{
    $stats = [];
    
    // Signup conversion
    $totalSignups = Capsule::table('mod_onboarding')->count();
    $firstOrder = Capsule::table('mod_onboarding')
        ->whereNotNull('first_order_at')
        ->count();
    
    $stats['signup_to_order_conversion'] = $totalSignups > 0 
        ? round(($firstOrder / $totalSignups) * 100, 2) 
        : 0;
    
    // Average time to first order
    $avgTimeQuery = Capsule::table('mod_onboarding')
        ->whereNotNull('first_order_at')
        ->selectRaw('AVG(TIMESTAMPDIFF(HOUR, started_at, first_order_at)) as avg_hours')
        ->first();
    
    $stats['avg_hours_to_first_order'] = $avgTimeQuery->avg_hours ?? 0;
    
    // Checklist completion
    $avgChecklistComplete = Capsule::table('mod_onboarding_checklist')
        ->selectRaw('AVG(CASE WHEN completed = 1 THEN 100 ELSE 0 END) as completion_rate')
        ->first();
    
    $stats['checklist_completion_rate'] = $avgChecklistComplete->completion_rate ?? 0;
    
    return $stats;
}
```

## Best Practices

1. **Reduce friction** - Minimize required fields for signup
2. **Immediate value** - Show value quickly after signup
3. **Clear progress** - Visual progress indicators
4. **Automated follow-ups** - Nudge abandoned signups
5. **Personalized emails** - Reference user's selections
6. **Offer help** - Easy access to support during onboarding
7. **Celebrate milestones** - Acknowledge completed steps
8. **Gather feedback** - Learn from drop-offs

## Common Pitfalls to Avoid

1. **Too many fields** - Signup form is too long
2. **No progress tracking** - User loses sense of progress
3. **Forgotten signups** - No follow-up on abandonment
4. **Confusing instructions** - Users don't know what to do
5. **Slow provisioning** - Delay between payment and service
6. **Missing communication** - No status updates
7. **No personalization** - Generic approach to all clients
8. **No feedback loop** - Not learning from onboarding data
