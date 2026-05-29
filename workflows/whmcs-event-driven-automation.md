# WHMCS Event-Driven Automation Workflow

## Overview
This workflow implements event-driven automation using WHMCS hooks system for real-time responses to system events.

## Prerequisites
- WHMCS v7.0+ installation
- PHP development knowledge
- Access to WHMCS hooks directory
- Basic understanding of hook system

## Step-by-Step Process

### Step 1: Understand Hook Architecture
WHMs hooks follow an event-subscriber pattern:
```
Event Occurs → Hook System → Registered Hook Functions → Actions Executed
```

### Step 2: Create Hook File
1. Navigate to WHMCS hooks directory:
   ```bash
   /includes/hooks/
   ```

2. Create new hook file:
   ```bash
   touch /includes/hooks/custom_events.php
   ```

3. Define hook handler:
   ```php
   <?php
   use WHMCS\View\Formatter\Date;

   /**
    * Custom Event-Driven Automation Hooks
    */
   ```

### Step 3: Register Hook Functions

#### Service Lifecycle Hooks
```php
// Service Created
add_hook('AfterServiceCreate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $userId = $vars['userid'];

    // Send welcome sequence
    sendWelcomeEmails($userId, $serviceId);

    // Provision external resources
    provisionExternalResources($vars);

    // Set up monitoring
    initializeServiceMonitoring($serviceId);
});

// Service Suspended
add_hook('ServiceSuspended', 1, function($vars) {
    $serviceId = $vars['serviceid'];

    // Notify client
    sendSuspensionNotice($serviceId);

    // Update external systems
    syncServiceStatus($serviceId, 'suspended');

    // Log event
    logActivity("Service {$serviceId} suspended", $vars['userid']);
});

// Service Terminated
add_hook('ServiceTerminated', 1, function($vars) {
    // Cleanup resources
    cleanupServiceResources($vars['serviceid']);

    // Update CRM
    updateCRMRecord($vars['userid'], 'terminated');

    // Archive data per retention policy
    archiveServiceData($vars['serviceid']);
});
```

#### Invoice Event Hooks
```php
// Invoice Created
add_hook('InvoiceCreated', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    $userId = $vars['userid'];

    // Apply auto-discounts
    checkAutoDiscounts($invoiceId);

    // Add promotional items
    checkPromotionalOffers($invoiceId);

    // Sync to accounting
    syncInvoiceToAccounting($invoiceId);
});

// Invoice Paid
add_hook('InvoicePaid', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];

    // Activate services
    activatePendingServices($invoiceId);

    // Update affiliate commissions
    creditAffiliateReferral($invoiceId);

    // Send receipt
    sendPaymentReceipt($invoiceId);

    // Create subscription renewal
    scheduleNextRenewal($invoiceId);
});

// Invoice Overdue
add_hook('InvoiceOverdue', 1, function($vars) {
    $daysOverdue = $vars['dues'];
    $invoiceId = $vars['invoiceid'];

    // Escalation based on overdue days
    if ($daysOverdue >= 30) {
        escalateToCollections($invoiceId);
    } elseif ($daysOverdue >= 14) {
        finalNoticeEmail($invoiceId);
    } elseif ($daysOverdue >= 7) {
        secondReminderEmail($invoiceId);
    }

    // Suspend services if overdue threshold met
    if ($daysOverdue >= getConfig('suspension_threshold_days')) {
        suspendOverdueServices($vars['userid']);
    }
});
```

#### Client Event Hooks
```php
// New Client Registration
add_hook('ClientAdd', 1, function($vars) {
    $clientId = $vars['userid'];

    // Create CRM record
    createCRMContact($clientId);

    // Set initial client tier
    setClientTier($clientId, 'standard');

    // Send welcome sequence
    triggerWelcomeEmailSequence($clientId);

    // Initialize tracking
    initializeClientAnalytics($clientId);

    // Assign welcome coupon
    grantWelcomeCredit($clientId);
});

// Client Login
add_hook('ClientLogin', 1, function($vars) {
    $clientId = $vars['userid'];

    // Update last login
    updateLastLogin($clientId);

    // Check for pending actions
    checkPendingActions($clientId);

    // Start session tracking
    startSessionAnalytics($clientId);

    // Check subscription health
    checkSubscriptionStatus($clientId);
});
```

### Step 4: Implement Conditional Logic
```php
add_hook('AfterServiceCreate', 1, function($vars) {
    // Load service details
    $service = Capsule::table('tblhosting')
        ->where('id', $vars['serviceid'])
        ->first();

    $product = Capsule::table('tblproducts')
        ->where('id', $service->packageid)
        ->first();

    // Conditional actions based on product type
    switch ($product->type) {
        case 'hosting':
            setupHostingAccount($vars['serviceid']);
            break;
        case 'reseller':
            setupResellerAccount($vars['serviceid']);
            break;
        case 'vps':
            provisionVPS($vars['serviceid']);
            break;
        case 'server':
            provisionDedicatedServer($vars['serviceid']);
            break;
    }
});
```

### Step 5: Create Async Processing
```php
add_hook('AfterServiceCreate', 1, function($vars) {
    // Queue resource-intensive tasks
    $queue = new WHMCS\Scheduling\QueuedTask\TaskQueue();

    $queue->add([
        'task' => 'ProvisionServiceResources',
        'data' => [
            'service_id' => $vars['serviceid'],
            'user_id' => $vars['userid'],
        ],
        'due' => time() // Immediate processing
    ]);

    $queue->add([
        'task' => 'SendWelcomeEmail',
        'data' => [
            'user_id' => $vars['userid'],
        ],
        'due' => strtotime('+1 minute')
    ]);
});
```

### Step 6: Add Error Handling
```php
add_hook('ServiceSuspended', 1, function($vars) {
    try {
        // Primary action
        notifyExternalSystem($vars['serviceid']);

        logActivity("Hook executed successfully", $vars['userid']);
    } catch (Exception $e) {
        logActivity("Hook error: " . $e->getMessage(), $vars['userid']);

        // Queue retry
        retryHook('ServiceSuspended', $vars, $e);

        // Alert admin
        sendAdminAlert("Hook failure in ServiceSuspended", $e);
    }
});
```

### Step 7: Implement Event Logging
```php
add_hook('ServiceSuspended', 1, function($vars) {
    $logEntry = [
        'event' => 'service_suspended',
        'service_id' => $vars['serviceid'],
        'user_id' => $vars['userid'],
        'timestamp' => date('Y-m-d H:i:s'),
        'triggered_by' => 'hook_system',
        'metadata' => json_encode($vars)
    ];

    Capsule::table('mod_event_log')->insert($logEntry);
});
```

## Advanced Patterns

### Event Correlation
Correlate multiple events for complex workflows:
```php
// Track state across events
class ServiceStateTracker {
    private static $state = [];

    public static function set($key, $value) {
        self::$state[$key] = $value;
    }

    public static function get($key) {
        return self::$state[$key] ?? null;
    }
}

add_hook('InvoiceCreated', 1, function($vars) {
    ServiceStateTracker::set('pending_invoice_' . $vars['userid'], true);
});

add_hook('InvoicePaid', 1, function($vars) {
    if (ServiceStateTracker::get('pending_invoice_' . $vars['userid'])) {
        // Invoice was pending, now paid - special handling
        applyFirstPaymentBonus($vars['userid']);
    }
});
```

### Webhook Integration
```php
add_hook('InvoicePaid', 1, function($vars) {
    $webhookUrl = 'https://api.example.com/webhooks/whmcs';
    $payload = [
        'event' => 'invoice.paid',
        'data' => [
            'invoice_id' => $vars['invoiceid'],
            'amount' => $vars['total'],
            'paid_at' => date('c')
        ]
    ];

    sendWebhook($webhookUrl, $payload);
});
```

## Testing Event Hooks

### Unit Testing
```php
class EventHookTest extends TestCase {
    public function testServiceSuspendedHook() {
        // Mock hook parameters
        $vars = [
            'serviceid' => 123,
            'userid' => 456
        ];

        // Execute hook
        $hook = new ServiceSuspendedHook();
        $hook->execute($vars);

        // Verify assertions
        $this->assertServiceSuspended($vars['serviceid']);
        $this->assertNotificationSent($vars['userid']);
    }
}
```

### Integration Testing
```bash
# Test specific hook
php whmcs_cli.php hook test AfterServiceCreate --service-id=123
```

## Performance Considerations

1. **Keep Hooks Fast**: Move heavy processing to queued tasks
2. **Minimize DB Queries**: Cache frequently accessed data
3. **Avoid Blocking**: Use async for external API calls
4. **Limit Hook Chains**: Prevent infinite loops with flags

## Related Workflows
- [WHMCS Automation Rules](./whmcs-automation-rules.md)
- [WHMCS Cron Automation](./whmcs-cron-automation.md)
- [WHMCS Queue Processing](./whmcs-queue-processing.md)