# WHMCS Grace Period Setup Documentation

## Overview

Grace periods provide customers with additional time to pay before services are suspended or terminated, improving customer experience while protecting revenue.

## Configuration

### Enable Grace Period

Navigate to: **Configuration > General Settings > Automation Settings**

```php
// Grace Period Configuration
$gracePeriodConfig = [
    'enabled' => true,
    'default_grace_days' => 7,

    // Grace Period Stages
    'stages' => [
        'reminder' => 3,              // Days before due
        'warning' => 0,               // Days after due
        'late_fee' => 3,              // Days after due
        'suspension' => 14,           // Days after due
        'termination' => 30           // Days after due
    ],

    // Extensions
    'allow_extensions' => true,
    'max_extension_days' => 7,
    'extensions_per_customer' => 2,

    // Notifications
    'notify_before_grace' => true,
    'notify_during_grace' => true,
    'notify_before_suspension' => true
];
```

## Grace Period Stages

### Standard Flow

```php
// Standard grace period workflow
$gracePeriodFlow = [
    'stages' => [
        0 => [
            'name' => 'Invoice Due',
            'description' => 'Invoice due date reached',
            'action' => 'none',
            'notify_customer' => true,
            'notify_admin' => false
        ],
        3 => [
            'name' => 'First Reminder',
            'description' => 'Invoice overdue by 3 days',
            'action' => 'reminder',
            'notify_customer' => true,
            'notify_admin' => false,
            'late_fee_apply' => false
        ],
        7 => [
            'name' => 'Grace Period Start',
            'description' => 'Entering grace period',
            'action' => 'grace_period',
            'notify_customer' => true,
            'notify_admin' => true,
            'late_fee_apply' => true,
            'suspend_services' => false
        ],
        14 => [
            'name' => 'Suspend Services',
            'description' => 'Services suspended',
            'action' => 'suspend',
            'notify_customer' => true,
            'notify_admin' => true,
            'suspend_services' => true,
            'preserve_data' => true
        ],
        30 => [
            'name' => 'Termination',
            'description' => 'Account terminated',
            'action' => 'terminate',
            'notify_customer' => true,
            'notify_admin' => true,
            'backup_data' => true,
            'remove_data' => false
        ]
    ]
];
```

### Product-Specific Grace

```php
// Grace period by product type
$productGracePeriods = [
    'hosting' => [
        'grace_days' => 7,
        'suspend_days' => 14,
        'terminate_days' => 30,
        'preserve_data_days' => 14
    ],
    'domain' => [
        'grace_days' => 30,            // Registry grace period
        'suspend_days' => 40,
        'terminate_days' => 45,
        'redemption_days' => 30
    ],
    'ssl' => [
        'grace_days' => 0,
        'suspend_days' => 0,
        'revoke_days' => 7
    ],
    'dedicated_server' => [
        'grace_days' => 3,
        'suspend_days' => 7,
        'terminate_days' => 14,
        'maintenance_fee_apply' => true
    ]
];
```

## Extension Management

### Allow Extensions

```php
// Customer extension settings
$extensionConfig = [
    'allow_customer_request' => true,
    'allow_admin_grant' => true,

    // Limits
    'max_extensions' => 2,
    'max_extension_days' => 7,
    'max_total_extension_days' => 14,

    // Approval
    'require_approval' => true,
    'auto_approve_for' => ['enterprise', 'vip'],
    'approval_threshold_days' => 3,

    // Conditions
    'require_payment_arrangement' => true,
    'require_credit_check' => true,
    'check_payment_history' => true,
    'minimum_payment_history_months' => 3
];
```

### Extension Request

```http
POST /billing/grace-period/extend
```

**Request Body:**

```json
{
  "invoice_id": "INV-12345",
  "extension_days": 7,
  "reason": "Temporary cash flow issue",
  "payment_commitment" => "Will pay by extended date"
}
```

**Response:**

```json
{
  "success": true,
  "extension": {
    "invoice_id": "INV-12345",
    "original_due_date": "2024-01-01",
    "new_due_date": "2024-01-08",
    "extension_days": 7,
    "approved_by" => "system",
    "extended_at": "2024-01-03T10:30:00Z"
  }
}
```

## Customer Management

### View Grace Period Status

```http
GET /clients/{client_id}/grace-period
```

**Response:**

```json
{
  "client_id": 12345,
  "status": "grace_period",
  "invoices_in_grace": [
    {
      "invoice_id": "INV-12345",
      "amount": 100.00,
      "due_date": "2024-01-01",
      "days_overdue": 5,
      "grace_days_remaining": 2,
      "next_action" => "suspension",
      "next_action_date": "2024-01-09"
    }
  ],
  "extensions_used" => 1,
  "extensions_remaining" => 1
}
```

### Request Extension

**Client Area > Billing > Invoice > Request Extension**

```
+------------------------------------------+
| Request Payment Extension                |
+------------------------------------------+
| Invoice: INV-12345                      |
| Amount Due: $100.00                     |
| Original Due Date: January 1, 2024      |
| Days Overdue: 5                        |
|                                          |
| Extension Requested: [7 days_________]  |
| New Due Date: January 8, 2024            |
|                                          |
| Reason: [Temporary cash flow issue____] |
|                                          |
| Payment Commitment:                      |
| [x] I commit to payment by new due date |
|                                          |
| Extensions Remaining: 1 of 2            |
|                                          |
| [Submit Request]                         |
+------------------------------------------+
```

## API Reference

### Get Grace Period Status

```http
GET /billing/grace-period/status/{invoice_id}
```

**Response:**

```json
{
  "invoice_id": "INV-12345",
  "status": "grace_period",
  "grace_period_start": "2024-01-01",
  "grace_period_end": "2024-01-08",
  "days_remaining": 3,
  "stages" => [
    {"stage" => "reminder", "reached" => true, "date" => "2024-01-03"},
    {"stage" => "grace", "reached" => true, "date" => "2024-01-01"},
    {"stage" => "suspension", "reached" => false, "scheduled_date" => "2024-01-09"}
  ],
  "extensions" => [
    {"days" => 7, "granted_by" => "system", "date" => "2024-01-03"}
  ]
}
```

### Grant Extension (Admin)

```http
POST /billing/grace-period/extend/{invoice_id}
```

**Request Body:**

```json
{
  "extension_days": 7,
  "reason" => "Good customer, payment arrangement made",
  "notify_customer" => true,
  "waive_late_fees" => false
}
```

### Revoke Extension

```http
POST /billing/grace-period/revoke/{invoice_id}
```

## Grace Period Notifications

### Notification Schedule

```php
// Grace period notifications
$graceNotifications = [
    'before_grace_starts' => [
        'enabled' => true,
        'days_before' => 3,
        'template' => 'grace_period_warning',
        'channels' => ['email']
    ],
    'grace_started' => [
        'enabled' => true,
        'template' => 'grace_period_started',
        'channels' => ['email', 'sms']
    ],
    'grace_reminder' => [
        'enabled' => true,
        'days_during' => [3, 5],
        'template' => 'grace_period_reminder',
        'channels' => ['email']
    ],
    'before_suspension' => [
        'enabled' => true,
        'days_before' => 3,
        'template' => 'suspension_warning',
        'channels' => ['email', 'sms']
    ],
    'suspension' => [
        'enabled' => true,
        'template' => 'services_suspended',
        'channels' => ['email']
    ]
];
```

### Email Templates

```php
// Grace period email templates
$graceEmailTemplates = [
    'grace_period_warning' => [
        'subject' => 'Payment Reminder - Invoice {invoice_num}',
        'variables' => ['invoice_num', 'amount', 'due_date', 'days_remaining']
    ],
    'grace_period_started' => [
        'subject' => 'Your account is in grace period',
        'variables' => ['days_remaining', 'payment_url', 'contact_support']
    ],
    'suspension_warning' => [
        'subject' => 'URGENT: Services will be suspended soon',
        'variables' => ['suspend_date', 'action_required', 'payment_url']
    ]
];
```

## Automation

### Grace Period Cron

```php
// Automation configuration
$graceAutomation = [
    'enabled' => true,
    'run_frequency' => 'daily',
    'run_time' => '06:00',
    'batch_size' => 100,

    // Actions
    'check_grace_status' => true,
    'apply_extensions' => true,
    'send_notifications' => true,
    'initiate_suspension' => true,
    'initiate_termination' => false,    // Usually manual

    // Notifications
    'batch_notifications' => true,
    'max_notifications_per_run' => 500
];
```

### Suspension Workflow

```php
// Automatic suspension workflow
$suspensionWorkflow = [
    'enabled' => true,
    'days_after_due' => 14,
    'prerequisites' => [
        'grace_period_expired' => true,
        'no_pending_extensions' => true,
        'notify_before' => true,
        'notify_before_days' => 3
    ],
    'actions' => [
        'suspend_services' => true,
        'preserve_data' => true,
        'revoke_access' => true,
        'send_notification' => true
    ],
    'post_suspension' => [
        'allow_renewal' => true,
        'allow_extension' => false,
        'restore_immediately_on_payment' => true
    ]
];
```

## Exceptions

### Customer Exemptions

```php
// Exempt from standard grace period
$graceExemptions = [
    'enterprise_customers' => [
        'enabled' => true,
        'group_ids' => ['enterprise', 'vip'],
        'grace_days' => 14,
        'suspend_days' => 30
    ],
    'good_standing' => [
        'enabled' => true,
        'payment_history_months' => 12,
        'on_time_percentage' => 95,
        'grace_days' => 14
    ],
    'first_invoice' => [
        'enabled' => true,
        'grace_days' => 14,
        'suspension_days' => 30
    ]
];
```

### Product Exemptions

```php
// Product-specific grace periods
$productGraceExceptions = [
    'domain_renewal' => [
        'grace_days' => 30,            // ICANN grace
        'redemption_days' => 30,        // After grace
        'no_suspension' => true
    ],
    'ssl_certificate' => [
        'grace_days' => 0,
        'immediate_revoke' => false,
        'revoke_after_days' => 7
    ],
    'reserved_hosting' => [
        'grace_days' => 0,
        'no_grace' => true,
        'immediate_suspension' => true
    ]
];
```

## Reporting

### Grace Period Report

```http
GET /billing/reports/grace-period
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_invoices_in_grace" => 100,
    "currently_in_grace" => 25,
    "exited_grace_paid" => 60,
    "exited_grace_suspended" => 10,
    "exited_grace_terminated" => 5
  },
  "average_grace_days" => 5,
  "by_product" => [
    {"product_type" => "hosting", "in_grace" => 15},
    {"product_type" => "domain", "in_grace" => 8},
    {"product_type" => "ssl", "in_grace" => 2}
  ]
}
```

## Customer Portal Display

### Grace Period Notice

**Client Area > Billing > Invoice**

```
+------------------------------------------------------------------+
|  INVOICE INV-12345                                               |
+------------------------------------------------------------------+
|  Status: IN GRACE PERIOD                                        |
|  Amount Due: $100.00                                            |
|  Due Date: January 1, 2024                                      |
|  Days Overdue: 5                                                |
|                                                                  |
|  +--------------------------------------------------------------+|
|  | GRACE PERIOD NOTICE                                          ||
|  |                                                             ||
|  | Your account is currently in grace period.                  ||
|  | Days remaining before suspension: 2                         ||
|  |                                                             ||
|  | Please make payment immediately to avoid service suspension.||
|  |                                                             ||
|  | [Pay Now] [Request Extension]                               ||
|  +--------------------------------------------------------------+|
+------------------------------------------------------------------+
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Grace period not starting | Automation not running | Check cron |
| Extension not applying | Limits reached | Check extension count |
| Suspension too early | Wrong days configured | Verify product settings |
| Notifications not sent | Template missing | Check email templates |

### Debug Commands

```bash
# Check grace period status
whmcscli graceperiod status --invoice_id=INV-12345

# View customers in grace
whmcscli graceperiod list --status=grace

# Grant extension
whmcscli graceperiod extend --invoice_id=INV-12345 --days=7

# Check grace automation
whmcscli graceperiod automation-status
```

## Best Practices

1. **Communicate clearly** - Display grace period status prominently
2. **Send reminders** - Multiple touchpoints before suspension
3. **Allow extensions** - For legitimate circumstances
4. **Track extensions** - Monitor abuse of extension policy
5. **Automate consistently** - Apply rules uniformly
6. **Document exceptions** - Keep records of all extensions

## See Also

- [Payment Terms](./whmcs-payment-terms.md)
- [Late Fee Rules](./whmcs-late-fee-rules.md)
- [Suspension Policy](./whmcs-suspension-policy.md)
