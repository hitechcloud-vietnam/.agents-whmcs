# WHMCS Dunning Settings

## Overview

Dunning (from German "dunning" meaning debt collection) is the automated process of managing overdue invoice collection through a series of reminder emails, escalating actions, and ultimately service suspension or termination. WHMCS provides comprehensive dunning configuration.

## Dunning Configuration

### Accessing Dunning Settings

**Configuration > Billing > Dunning Management**

### Basic Configuration

```php
[
    'enabled' => true,
    'days_before_first_reminder' => 0,     // Days after due date
    'max_reminders' => 4,
    'escalation_actions' => [
        'suspension_days' => 14,
        'termination_days' => 28
    ]
]
```

## Dunning Schedule

### Reminder Sequence

```php
// Configure reminder timeline
[
    'reminders' => [
        [
            'day' => 0,                    // Due date
            'type' => 'due_date',
            'template' => 'invoice_due',
            'enabled' => true
        ],
        [
            'day' => 3,
            'type' => 'reminder',
            'template' => 'invoice_reminder_1',
            'enabled' => true
        ],
        [
            'day' => 7,
            'type' => 'reminder',
            'template' => 'invoice_reminder_2',
            'enabled' => true
        ],
        [
            'day' => 10,
            'type' => 'warning',
            'template' => 'suspension_warning',
            'enabled' => true
        ],
        [
            'day' => 14,
            'type' => 'suspension',
            'action' => 'suspend_service',
            'enabled' => true
        ]
    ]
]
```

### Custom Schedule

```php
// Extended dunning schedule
[
    'day_0' => ['email' => 'invoice_due', 'action' => 'none'],
    'day_3' => ['email' => 'overdue_notice_1', 'action' => 'none'],
    'day_5' => ['email' => 'second_reminder', 'action' => 'none'],
    'day_7' => ['email' => 'final_warning', 'action' => 'none'],
    'day_10' => ['email' => 'suspension_imminent', 'action' => 'email'],
    'day_14' => ['email' => 'suspended', 'action' => 'suspend'],
    'day_21' => ['email' => 'termination_warning', 'action' => 'email'],
    'day_28' => ['email' => 'terminated', 'action' => 'terminate']
]
```

## Email Templates

### Default Dunning Templates

| Template | Description | Default Day |
|----------|-------------|-------------|
| Invoice Due | Sent on due date | Day 0 |
| Invoice Reminder 1 | First reminder | Day 3 |
| Invoice Reminder 2 | Second reminder | Day 7 |
| Suspension Warning | Before suspension | Day 10 |
| Account Suspended | After suspension | Day 14 |
| Termination Warning | Before termination | Day 21 |
| Account Terminated | After termination | Day 28 |

### Template Variables

```smarty
<!-- Dunning email variables -->
{$invoice_number}
{$invoice_amount}
{$amount_due}
{$due_date}
{$days_overdue}
{$client_name}
{$client_email}
{$payment_link}
{$service_name}
```

## Escalation Actions

### Suspension Configuration

```php
// Auto-suspend overdue services
[
    'auto_suspend' => true,
    'suspend_days' => 14,               // Days after due date
    'suspend_on_grace_period_end' => true,
    'suspend_email' => 'service_suspended',
    'suspend_reason' => 'Overdue Invoice',
    'days_before_suspension_email' => 3,
    'include_unpaid_invoices' => true
]
```

### Termination Configuration

```php
// Auto-terminate services
[
    'auto_terminate' => true,
    'terminate_days' => 28,             // Days after due date
    'terminate_on_grace_period_end' => true,
    'terminate_email' => 'service_terminated',
    'preserve_data_days' => 7,          // Keep data before deletion
    'require_confirmation' => false,
    'terminated_status' => 'Terminated'
]
```

### Additional Escalations

```php
// Other escalation actions
[
    'escalations' => [
        ['day' => 7, 'action' => 'add_late_fee', 'fee' => 5.00],
        ['day' => 14, 'action' => 'restrict_portal', 'limit' => 'view_only'],
        ['day' => 21, 'action' => 'notify_sales', 'email' => 'sales@example.com']
    ]
]
```

## Late Fees

### Automatic Late Fees

```php
// Configure late fees
[
    'late_fee_enabled' => true,
    'late_fee_amount' => 5.00,
    'late_fee_type' => 'fixed',         // fixed, percentage
    'late_fee_percentage' => 2,
    'apply_after_days' => 7,
    'maximum_late_fees' => 25.00,
    'recurring_fee' => false,
    'fee_invoice_group' => 'Late Fees'
]
```

### Late Fee Example

```php
// Fixed fee
$invoiceAmount = 100.00;
$lateFee = 5.00;
$totalDue = 105.00;

// Percentage fee
$lateFeePercentage = 2;
$lateFee = 100.00 * 0.02;  // $2.00
$totalDue = 102.00;
```

## Dunning Rules

### Per-Product Rules

```php
// Custom dunning per product
[
    'product_id' => 1,
    'custom_dunning' => true,
    'reminder_days' => [0, 5, 10, 15],
    'suspend_days' => 20,
    'terminate_days' => 35
]
```

### Per-Client Rules

```php
// VIP client leniency
[
    'client_id' => 123,
    'extend_grace_days' => 14,     // Extra 14 days
    'max_reminders' => 6,
    'skip_suspension' => true
]
```

## Dunning Exclusion

### Exclude from Dunning

```php
// Skip certain invoices/clients
[
    'exclude_invoice_types' => [
        'proforma',
        'quote'
    ],
    'exclude_client_groups' => [
        'premium',
        'enterprise'
    ],
    'exclude_amount_min' => 500.00,
    'exclude_amount_max' => 50000.00
]
```

## Dunning Reports

### Dunning Effectiveness

**Reports > Billing > Dunning Report**

```php
// Dunning statistics
[
    'period' => 'May 2024',
    'invoices_sent_to_dunning' => 150,
    'recovered_invoices' => 120,
    'recovery_rate' => 80,
    'total_recovered' => 15000.00,
    'suspended_services' => 15,
    'terminated_services' => 5
]
```

### Reminder Performance

```php
// Per-reminder effectiveness
[
    'reminder_1' => ['sent' => 150, 'recovered' => 45, 'rate' => 30],
    'reminder_2' => ['sent' => 105, 'recovered' => 40, 'rate' => 38],
    'reminder_3' => ['sent' => 65, 'recovered' => 20, 'rate' => 31],
    'reminder_4' => ['sent' => 45, 'recovered' => 15, 'rate' => 33]
]
```

## Manual Dunning

### Manual Override

```php
// Admin manual dunning control
[
    'allow_manual_dunning' => true,
    'allow_skip_reminder' => true,
    'allow_extend_grace' => true,
    'require_reason' => true,
    'log_all_actions' => true
]
```

### Skip Reminder

```php
// Skip next reminder
[
    'invoice_id' => 5678,
    'skip_next_reminder' => true,
    'skip_until' => '2024-05-20',
    'reason' => 'Client contact confirmed payment'
]
```

## Dunning Automation

### Cron Configuration

```bash
# Dunning cron runs with main cron
* * * * * php -q /whmcs/crons/autocron.php

# Or dedicated dunning cron
0 8 * * * php -q /whmcs/crons/dunning.php
```

### Automation Logic

```php
// Daily dunning process
1. Query overdue invoices (due_date < today)
2. For each invoice:
   a. Check dunning rules
   b. Determine current stage
   c. Send appropriate reminder
   d. Apply late fees if applicable
   e. Trigger suspension if threshold reached
   f. Trigger termination if threshold reached
3. Log all actions
4. Generate report
```

## Hooks

```php
// Hook: PreDunningReminder
add_hook('PreDunningReminder', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['reminder_day']
    // $vars['template']
    
    // Modify reminder or prevent it
    return ['send' => true, 'template' => $vars['template']];
});

// Hook: DunningReminderSent
add_hook('DunningReminderSent', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['reminder_number']
    // $vars['email_to']
});

// Hook: PreSuspendService
add_hook('PreSuspendService', 1, function($vars) {
    // $vars['serviceid']
    // $vars['invoiceid']
    // $vars['reason']
});
```

## Best Practices

1. **Start gentle**: Begin with friendly reminders
2. **Clear escalation**: Make consequences clear
3. **Personalize messages**: Reference specific invoice
4. **Offer payment options**: Provide easy payment links
5. **Monitor effectiveness**: Adjust schedule based on data

## Related Documentation

- [Invoice Reminders](./whmcs-invoice-reminders.md)
- [Late Fees](./whmcs-late-fees.md)
- [Service Suspension](./whmcs-service-suspension.md)
- [Service Termination](./whmcs-service-termination.md)