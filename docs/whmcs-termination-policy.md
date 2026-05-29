# WHMCS Termination Policy Documentation

## Overview

Termination policies define when and how services are terminated for non-payment, including data handling, final notices, and cleanup procedures.

## Configuration

### Enable Termination

Navigate to: **Configuration > System Settings > Automation Settings**

```php
// Termination Configuration
$terminationConfig = [
    'enabled' => true,

    // Timing
    'days_after_suspension' => 30,
    'minimum_days_after_suspend' => 14,
    'immediate_termination' => false,

    // Pre-Termination
    'require_suspension' => true,
    'notify_before_termination' => true,
    'notify_days_before' => [7, 3, 1],
    'final_notice_enabled' => true,

    // Termination Actions
    'terminate_service' => true,
    'delete_data' => true,
    'delete_data_days' => 7,          // After termination
    'backup_before_delete' => true,
    'backup_retention_days' => 30,
    'release_domains' => true,
    'cancel_registrations' => false,

    // Final Billing
    'final_invoice' => true,
    'write_off_balance' => true,
    'report_to_credit_agency' => false
];
```

## Termination Rules

### Standard Termination

```php
// Standard termination rule
$standardTermination = [
    'name' => 'Standard Termination',
    'days_after_suspend' => 30,

    // Pre-conditions
    'suspension_completed' => true,
    'no_active_tickets' => false,     // Don't wait for tickets
    'no_pending_refunds' => true,

    // Warning Sequence
    'warnings' => [
        ['days_before' => 7, 'template' => 'termination_warning_7'],
        ['days_before' => 3, 'template' => 'termination_warning_3'],
        ['days_before' => 1, 'template' => 'termination_warning_1']
    ],

    // Termination Actions
    'actions' => [
        'terminate_service' => true,
        'backup_data' => true,
        'delete_after_days' => 7,
        'release_domains' => false,
        'cancel_pending_orders' => true,
        'send_termination_notice' => true
    ],

    // Data Handling
    'backup_before_termination' => true,
    'backup_location' => 's3://backups/terminated/',
    'retention_days' => 30,
    'auto_delete_after_retention' => true
];
```

### Product-Specific Termination

```php
// Termination rules by product
$productTerminations = [
    'hosting' => [
        'terminate_days_after_suspend' => 30,
        'backup_before_terminate' => true,
        'delete_data_days' => 14,
        'release_associated_domains' => false,
        'delete_databases' => true,
        'delete_email_accounts' => true,
        'delete_files' => true
    ],
    'domain' => [
        'terminate_days_after_suspend' => 45,
        'expiration_handling' => true,
        'registry_grace_period' => 30,
        'redemption_period' => 30,
        'release_to_registry' => true
    ],
    'reseller' => [
        'terminate_days_after_suspend' => 21,
        'suspend_child_accounts_first' => true,
        'migrate_child_accounts' => false,
        'delete_parent_and_children' => true
    ],
    'dedicated_server' => [
        'terminate_days_after_suspend' => 7,
        'immediate_data_wipe' => true,
        'noc_ticket_required' => true,
        'wipe_certificate' => true
    ]
];
```

## Termination Process

### Termination Workflow

```
1. Suspension Active (14+ days)
2. Warning Sequence Initiated (7 days before)
   - Warning 1: 7 days before
   - Warning 2: 3 days before
   - Warning 3: 1 day before
3. Final Notice (Termination Day)
4. Termination Day Reached
5. Pre-Termination Checks
   - Extension active? -> Wait
   - Support ticket? -> Flag for review
   - Active services? -> Document
6. Create Final Backup
7. Execute Termination
8. Cancel Associated Services
9. Send Termination Confirmation
10. Update Database
11. Cleanup & Data Deletion
```

### Termination Actions

```php
// Actions performed on termination
$terminationActions = [
    // Immediate Actions
    'terminate_account' => [
        'command' => 'cpanel_terminate',
        'wait_for_completion' => true,
        'timeout' => 120
    ],
    'delete_all_files' => true,
    'drop_all_databases' => true,
    'remove_email_accounts' => true,
    'revoke_ssl_certificates' => true,

    // Documentation
    'create_termination_record' => true,
    'log_final_usage' => true,
    'generate_final_invoice' => true,

    // Post-Termination
    'send_termination_email' => true,
    'archive_customer_data' => true,
    'update_customer_status' => 'terminated',
    'remove_from_automation' => true
];
```

## Data Handling

### Backup Before Termination

```php
// Pre-termination backup
$preTerminationBackup = [
    'enabled' => true,
    'backup_location' => 's3://backups/terminated/',
    'backup_includes' => [
        'website_files' => true,
        'databases' => true,
        'email_mails' => true,
        'configuration' => true,
        'logs' => true
    ],
    'compression' => 'gzip',
    'encryption' => true,
    'retention_days' => 30,
    'auto_delete' => true
];
```

### Data Deletion

```php
// Data deletion rules
$dataDeletion = [
    'enabled' => true,
    'delay_days' => 7,                // Delete 7 days after termination
    'deletion_methods' => [
        'files' => 'secure_delete',   // overwrite before delete
        'databases' => 'drop',         // drop tables
        'email' => 'delete_maildir',   // remove maildir
        'backups' => 'retain'          // keep backup for retention period
    ],
    'verify_deletion' => true,
    'deletion_certificate' => true,    // Generate certificate
    'customer_notification' => true
];
```

## API Reference

### Terminate Service

```http
POST /services/{service_id}/terminate
```

**Request Body:**

```json
{
  "reason" => "non_payment",
  "invoice_id" => "INV-12345",
  "backup_before_terminate" => true,
  "notify_customer" => true,
  "delete_data_days" => 7,
  "termination_type" => "immediate"
}
```

**Response:**

```json
{
  "success" => true,
  "service_id" => 67890,
  "terminated_at" => "2024-01-15T10:30:00Z",
  "termination_reason" => "non_payment",
  "backup_created" => true,
  "backup_id" => "BACKUP-12345",
  "data_deletion_scheduled" => "2024-01-22",
  "termination_id" => "TERM-12345"
}
```

### Schedule Termination

```http
POST /services/{service_id}/terminate/schedule
```

**Request Body:**

```json
{
  "scheduled_date" => "2024-02-01",
  "reason" => "non_payment",
  "backup_first" => true,
  "notify_customer" => true
}
```

### Cancel Termination

```http
POST /services/{service_id}/terminate/cancel
```

**Request Body:**

```json
{
  "reason" => "Customer payment received",
  "restore_service" => true
}
```

### Get Termination Status

```http
GET /services/{service_id}/termination
```

**Response:**

```json
{
  "service_id" => 67890,
  "status" => "termination_scheduled",
  "scheduled_termination_date" => "2024-02-01",
  "termination_reason" => "non_payment",
  "warnings_sent" => 2,
  "warnings_remaining" => 1,
  "next_warning_date" => "2024-01-29",
  "backup_available" => true,
  "backup_id" => "BACKUP-12345"
}
```

## Final Invoice

### Generate Final Invoice

```php
// Final invoice for terminated service
$finalInvoice = [
    'enabled' => true,
    'generate_on_termination' => true,
    'include_items' => [
        'outstanding_balance' => true,
        'termination_fee' => false,
        'pro_rata_credits' => true,
        'setup_fees_non_refundable' => true
    ],
    'adjustments' => [
        'credit_applied' => true,
        'write_off_small_balances' => true,
        'write_off_threshold' => 1.00
    ],
    'status' => 'due_immediately'
];
```

### Invoice Line Items

```json
{
  "items": [
    {
      "description" => "Service charges through termination date",
      "amount" => 50.00
    },
    {
      "description" => "Credit for unused portion",
      "amount" => -25.00,
      "type" => "credit"
    }
  ],
  "subtotal" => 25.00,
  "credits_applied" => 0,
  "total" => 25.00,
  "write_off_eligible" => true,
  "write_off_threshold" => 1.00
}
```

## Notifications

### Termination Notifications

```php
// Termination notification schedule
$terminationNotifications = [
    'warning_7_days' => [
        'enabled' => true,
        'days_before' => 7,
        'template' => 'termination_warning_7',
        'channels' => ['email', 'sms']
    ],
    'warning_3_days' => [
        'enabled' => true,
        'days_before' => 3,
        'template' => 'termination_warning_3',
        'channels' => ['email', 'sms']
    ],
    'warning_1_day' => [
        'enabled' => true,
        'days_before' => 1,
        'template' => 'termination_warning_1',
        'channels' => ['email', 'sms']
    ],
    'termination_executed' => [
        'enabled' => true,
        'template' => 'service_terminated',
        'channels' => ['email']
    ],
    'data_deletion_warning' => [
        'enabled' => true,
        'days_before_deletion' => 3,
        'template' => 'data_deletion_warning',
        'channels' => ['email']
    ]
];
```

## Exceptions

### Exempt from Termination

```php
// Termination exemptions
$terminationExemptions = [
    'enterprise_customers' => [
        'enabled' => true,
        'groups' => ['enterprise'],
        'never_terminate' => false,
        'require_cfo_approval' => true,
        'escalate_to_collections' => true
    ],
    'active_disputes' => [
        'enabled' => true,
        'check_tickets' => true,
        'ticket_status_check' => ['open', 'pending_customer'],
        'pause_termination' => true
    ],
    'government_nonprofit' => [
        'enabled' => true,
        'never_terminate' => true,
        'flag_for_review' => true,
        'require_monthly_review' => true
    ]
];
```

### Manual Override

```php
// Override automatic termination
$manualOverride = [
    'enabled' => true,
    'require_reason' => true,
    'reason_options' => [
        'payment_arrangement',
        'service_credit_issued',
        'dispute_resolved',
        'executive_decision',
        'other'
    ],
    'extend_days' => 30,              // Extend instead of cancelling
    'notify_on_override' => true,
    'log_override' => true
];
```

## Reporting

### Termination Report

```http
GET /billing/reports/terminations
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_terminations" => 25,
    "auto_terminations" => 20,
    "manual_terminations" => 5,
    "recoveries" => 5,
    "data_deleted" => 15
  },
  "revenue_impact" => {
    "lost_mrr" => 2500.00,
    "termination_fees_collected" => 0,
    "recovery_rate" => 20
  },
  "by_product" => [
    {"product" => "hosting", "terminations" => 15},
    {"product" => "reseller", "terminations" => 7},
    {"product" => "vps", "terminations" => 3}
  ],
  "reasons" => [
    {"reason" => "non_payment", "count" => 20},
    {"reason" => "customer_request", "count" => 3},
    {"reason" => "fraud", "count" => 2}
  ]
}
```

## Customer Communication

### Termination Notice Template

```
+------------------------------------------------------------------+
|  IMPORTANT: Service Termination Notice                             |
+------------------------------------------------------------------+
|                                                                  |
|  Dear Customer,                                                  |
|                                                                  |
|  Your service is scheduled for termination on February 1, 2024   |
|  due to non-payment.                                            |
|                                                                  |
|  Service: Premium Hosting                                       |
|  Termination Date: February 1, 2024                             |
|                                                                  |
|  WHAT YOU SHOULD KNOW:                                          |
|                                                                  |
|  - Your data will be backed up and retained for 30 days        |
|  - After 30 days, all data will be permanently deleted         |
|  - You can still restore your service by making payment         |
|                                                                  |
|  TO AVOID TERMINATION:                                         |
|                                                                  |
|  Pay your invoice now at: [Pay Invoice - $100.00]              |
|                                                                  |
|  Need help? Contact us before the termination date.             |
+------------------------------------------------------------------+
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Termination not executing | Cron not running | Check automation |
| Data lost prematurely | Deletion too fast | Increase retention |
| Domain released incorrectly | Registry rules | Verify release settings |
| Customer disputes | Communication unclear | Update notices |

### Debug Commands

```bash
# Check termination status
whmcscli service termination --service_id=67890

# List pending terminations
whmcscli service termination-pending

# Cancel termination
whmcscli service terminate-cancel --service_id=67890

# View terminated services
whmcscli service list --status=terminated
```

## Best Practices

1. **Communicate clearly** - Multiple warnings before termination
2. **Backup first** - Always backup before termination
3. **Set retention** - Keep data for reasonable period
4. **Make recovery easy** - Allow restoration after payment
5. **Document everything** - Maintain audit trail
6. **Review patterns** - Analyze termination reasons

## See Also

- [Suspension Policy](./whmcs-suspension-policy.md)
- [Grace Period](./whmcs-grace-period.md)
- [Early Termination](./whmcs-early-termination.md)
