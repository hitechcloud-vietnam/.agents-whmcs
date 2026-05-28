# WHMCS Cron Events Reference

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `hooks-reference`, `database-indexing-guide`

## Overview

WHMCS uses a cron job system to automate recurring tasks. The WHMCS cron runs via `cron.php` and can be configured with various modules and schedules.

## Cron Job Setup

### Basic Cron Command

```bash
# Standard cron command (run every 5 minutes recommended)
*/5 * * * * /usr/bin/php -q /path/to/whmcs/crons/cron.php

# Alternative with specific settings
php -q /home/user/public_html/whmcs/crons/cron.php

# Using wget (not recommended for production)
wget -q -O /dev/null https://yourwhmcs.com/crons/cron.php

# Using curl
curl -s -o /dev/null https://yourwhmcs.com/crons/cron.php
```

### Cron URL Security

```php
// In crons/config.php
<?php
$cronSecurityKey = 'your-secure-random-key-here';
```

```bash
# With security key
*/5 * * * * /usr/bin/php -q /path/to/whmcs/crons/cron.php?cron_security_key=your-secure-random-key-here
```

## Cron Configuration

```php
// crons/config.php

return [
    'cron_secret' => 'your-secret-key',

    // Module-specific settings
    'module' => [
        'automatically_provision_suspended_services' => false,
        'use_advisory_locking' => true,
        'lock_timeout' => 300,
    ],

    // Email queue settings
    'email_queue' => [
        'batch_size' => 50,
        'max_retries' => 3,
    ],

    // Log settings
    'logging' => [
        'enabled' => true,
        'retention_days' => 30,
    ],
];
```

## Built-in Cron Modules

### Domain Renewal Sync

```bash
# Add to cron: Domain Transfer Sync Module
/usr/bin/php -q /path/to/whmcs/crons/domainssync.php
```

```php
// Configuration in Configuration.php
$domain_sync_enabled = true;
$domain_sync_days = 30; // Days before expiry to process
$domain_renewal_mode = 'notify'; // 'notify', 'autorenew', 'queue'
```

### Invoice Creation

```php
// Controlled via WHMCS Admin > Setup > Automation Settings

/*
 * Automate Invoice Generation
 *
 * Determines when the automation creates
 * invoices for upcoming renewals.
 *
 * 0 = Disabled
 * 1-28 = Days before due date
 */
$invoiceGenerationDays = 7;
```

### Payment Reminders

```php
// In Configuration.php or Admin Settings

// Invoice reminder email schedules
$invoiceReminderDays = [
    1 => 'first_reminder',
    4 => 'second_reminder',
    7 => 'third_reminder',
    14 => 'fourth_reminder',
];

// Payment overdue actions
$overdueActions = [
    'suspend_services' => true,
    'suspend_days' => 14,
    'terminate_days' => 45,
];
```

### Service Module Cron

```bash
# Service Usage/Stats Sync
/usr/bin/php -q /path/to/whmcs/crons/usagebilling.php
```

```php
// Configuration for usage-based billing
$usageBillingEnabled = true;
$usageBillingCycle = 'monthly'; // 'monthly', 'daily'
$usageBillingProration = true;
```

## Custom Cron Modules

### Creating Custom Cron Module

```php
<?php
// crons/modules/custom/example_cron.php

/**
 * Custom Cron Module
 *
 * @package WHMCS\Cron\Modules
 */

namespace WHMCS\Cron\Modules;

class ExampleCron
{
    protected $cronData = [];

    /**
     * Initialize the cron module
     */
    public function __construct()
    {
        // Load dependencies
    }

    /**
     * Execute the cron task
     *
     * @param array $cronData Cron configuration data
     * @return array Result of execution
     */
    public function execute(array $cronData): array
    {
        $this->cronData = $cronData;
        $startTime = microtime(true);

        try {
            // Your cron logic here
            $processed = $this->processCustomTask();

            return [
                'success' => true,
                'processed' => $processed,
                'duration' => microtime(true) - $startTime,
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage(),
                'duration' => microtime(true) - $startTime,
            ];
        }
    }

    /**
     * Custom task processing
     */
    protected function processCustomTask(): int
    {
        // Get data to process
        $items = Capsule::table('tblcustom')
            ->where('status', 'pending')
            ->where('next_run', '<=', date('Y-m-d H:i:s'))
            ->get();

        $processed = 0;
        foreach ($items as $item) {
            // Process each item
            $this->processItem($item);
            $processed++;
        }

        return $processed;
    }

    /**
     * Process individual item
     */
    protected function processItem($item): void
    {
        // Update next run time
        Capsule::table('tblcustom')
            ->where('id', $item->id)
            ->update([
                'last_run' => date('Y-m-d H:i:s'),
                'next_run' => date('Y-m-d H:i:s', strtotime('+1 day')),
            ]);

        // Log activity
        logActivity("Processed custom cron item: {$item->id}");
    }

    /**
     * Required: Return module information
     */
    public static function getInfo(): array
    {
        return [
            'name' => 'Example Custom Cron',
            'description' => 'Processes custom scheduled tasks',
            'frequency' => [
                'times' => ['*/5', '*', '*', '*', '*'],
                'description' => 'Every 5 minutes',
            ],
        ];
    }
}
```

### Registering Custom Cron Module

```php
// Add to crons/modules.php

return [
    'system' => [
        'AutoClientSync',
        'AffiliateCommission',
        'DomainTransferSync',
        'DomainRenewalSync',
        'EmailMarketer',
        'InvoiceAccept',
        'InvoiceCreation',
        'ModuleCommandQueue',
        'OverdueHook',
        'PaymentReminder',
        'PendingOrder',
        'ServerUsageSync',
        'SuspendServices',
        'TerminateInactive',
        'TicketEscalations',
        'TicketAutoClose',
        'UsageBilling',
    ],
    'custom' => [
        'ExampleCron', // Your custom module
    ],
];
```

## Hook Integration with Cron

### Using Hooks with Cron

```php
// includes/hooks/cron_hooks.php

// Before cron execution
add_hook('CronJobsHead', 1, function($vars) {
    logActivity('Cron job starting: ' . date('Y-m-d H:i:s'));
});

// After cron completion
add_hook('CronJobsFoot', 1, function($vars) {
    logActivity('Cron job completed: ' . date('Y-m-d H:i:s'));

    // Send notification if cron took too long
    if ($vars['executionTime'] > 300) {
        sendCronSlowAlert($vars['executionTime']);
    }
});

// Before specific module execution
add_hook('CronJobRunning', 1, function($vars) {
    if ($vars['module'] === 'InvoiceCreation') {
        // Pre-processing for invoice creation
    }
});

// After specific module execution
add_hook('CronJobComplete', 1, function($vars) {
    if ($vars['module'] === 'CustomModule') {
        logActivity("Custom module processed {$vars['processed']} items");
    }
});
```

## Event Scheduling Patterns

### Standard Cron Expressions

```bash
# Every minute
* * * * *

# Every 5 minutes
*/5 * * * *

# Every 15 minutes
*/15 * * * *

# Every 30 minutes
*/30 * * * *

# Every hour
0 * * * *

# Every day at midnight
0 0 * * *

# Every day at 3 AM
0 3 * * *

# Every Monday at 9 AM
0 9 * * 1

# First day of every month at midnight
0 0 1 * *

# Every 6 hours
0 */6 * * *
```

### WHMCS Recommended Schedule

```bash
# Main cron - every 5 minutes (required)
*/5 * * * * /usr/bin/php -q /home/user/public_html/whmcs/crons/cron.php

# Domain sync - every hour (recommended)
0 * * * * /usr/bin/php -q /home/user/public_html/whmcs/crons/domainssync.php

# Usage billing - daily at 1 AM
0 1 * * * /usr/bin/php -q /home/user/public_html/whmcs/crons/usagebilling.php
```

## Cron Tasks and Their Frequency

| Task | Default Frequency | Description |
|------|-------------------|-------------|
| Auto Suspension | Every 12 hours | Suspends overdue services |
| Auto Termination | Every 12 hours | Terminates long-overdue services |
| Invoice Creation | Daily | Generates renewal invoices |
| Payment Reminders | Daily | Sends overdue payment emails |
| Pending Orders | Every 5 minutes | Auto-provision pending orders |
| Ticket Escalations | Every 15 minutes | Escalates aged tickets |
| Ticket Auto-Close | Every hour | Closes resolved tickets |
| Domain Sync | Every 6 hours | Syncs domain registration status |
| Usage Billing | Daily | Calculates usage-based charges |
| Affiliate Payouts | Weekly | Processes affiliate commissions |
| Server Usage Stats | Every 6 hours | Syncs bandwidth/disk usage |

## Monitoring Cron Execution

### Cron Log Location

```
/storage/logs/cron/
├── cron-2026-05-28.log
├── cron-2026-05-27.log
└── ...
```

### Log Format

```log
[2026-05-28 14:00:01] Cron: Starting execution
[2026-05-28 14:00:01] Module: InvoiceCreation - Starting
[2026-05-28 14:00:03] Module: InvoiceCreation - Generated 15 invoices
[2026-05-28 14:00:03] Module: InvoiceCreation - Completed in 2.1s
[2026-05-28 14:00:03] Module: PaymentReminder - Starting
[2026-05-28 14:00:05] Module: PaymentReminder - Sent 8 reminders
[2026-05-28 14:00:05] Module: PaymentReminder - Completed in 2.0s
[2026-05-28 14:00:05] Module: PendingOrder - Starting
[2026-05-28 14:00:06] Module: PendingOrder - Processed 3 orders
[2026-05-28 14:00:06] Module: PendingOrder - Completed in 1.0s
[2026-05-28 14:00:06] Cron: Execution completed in 5.2s
```

### Custom Logging

```php
// In custom cron module
use WHMCS\Module\Cron;

class CustomCron
{
    protected function log(string $message, string $level = 'info'): void
    {
        $logFile = ROOTDIR . '/storage/logs/cron/custom-' . date('Y-m-d') . '.log';
        $timestamp = date('Y-m-d H:i:s');
        $logEntry = "[{$timestamp}] [{$level}] {$message}\n";

        file_put_contents($logFile, $logEntry, FILE_APPEND);
    }
}
```

## Troubleshooting Cron Issues

### Common Issues

```php
// Issue: Cron not executing
// Solution: Check cron security key
$cronSecurityKey = 'your-correct-key'; // Must match admin setting

// Issue: Module not running
// Solution: Verify module is registered in modules.php
'custom' => ['YourModuleName'];

// Issue: Timeout errors
// Solution: Increase PHP timeout
// In crons/config.php
ini_set('max_execution_time', 600);

// Issue: Memory errors
// Solution: Increase memory limit
ini_set('memory_limit', '512M');
```

### Testing Cron Modules

```bash
# Test specific module
/usr/bin/php -q /path/to/whmcs/crons/cron.php?module=InvoiceCreation

# Test with security key
/usr/bin/php -q /path/to/whmcs/crons/cron.php?cron_security_key=your-key

# Verbose output
/usr/bin/php -d display_errors=1 -d error_reporting=E_ALL /path/to/whmcs/crons/cron.php
```

## Best Practices

1. **Run Cron Frequently**: Every 5 minutes is recommended
2. **Use Security Key**: Always secure cron with a secret key
3. **Monitor Logs**: Regularly check cron execution logs
4. **Separate Heavy Tasks**: Run heavy modules on separate schedules
5. **Handle Failures Gracefully**: Implement retry logic
6. **Database Optimization**: Ensure proper indexing for cron queries
7. **Timeout Management**: Set appropriate timeouts for long-running tasks

## Related Documentation

- [Hooks Reference](hooks-reference.md)
- [Database Indexing Guide](database-indexing-guide.md)
- [Cron Job Reference](cron-job-reference.md)
