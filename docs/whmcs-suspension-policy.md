# WHMCS Suspension Policy Documentation

## Overview

Suspension policies define when and how services are suspended for non-payment, balancing customer communication with revenue protection.

## Configuration

### Enable Suspension

Navigate to: **Configuration > System Settings > Automation Settings**

```php
// Suspension Configuration
$suspensionConfig = [
    'enabled' => true,

    // Timing
    'days_after_due' => 14,
    'days_after_suspend' => 30,      // Before termination
    'immediate_suspend' => false,

    // Pre-Suspension
    'require_grace_period' => true,
    'grace_period_days' => 7,
    'notify_before_suspend' => true,
    'notify_days_before' => 3,

    // Suspension Process
    'suspend_all_services' => true,
    'preserve_data' => true,
    'revoke_access' => true,
    'suspend_emails' => true,
    'suspend_databases' => true,

    // Post-Suspension
    'allow_login' => false,
    'show_suspension_notice' => true,
    'auto_unsuspend_on_payment' => true
];
```

## Suspension Rules

### Standard Suspension

```php
// Standard suspension rule
$standardSuspension = [
    'name' => 'Standard Suspension',
    'days_after_invoice_due' => 14,

    // Pre-conditions
    'grace_period_completed' => true,
    'no_pending_extensions' => true,
    'payment_reminders_sent' => 3,

    // Suspension Actions
    'actions' => [
        'suspend_hosting' => true,
        'suspend_database' => true,
        'suspend_email' => true,
        'suspend_subdomains' => true,
        'revoke_ftp_access' => true,
        'show_suspension_page' => true,
        'send_suspension_notice' => true
    ],

    // Data Preservation
    'preserve_website' => true,
    'preserve_database' => true,
    'preserve_emails' => true,
    'retention_days' => 30,

    // Restoration
    'auto_restore_on_payment' => true,
    'restore_delay_hours' => 1
];
```

### Product-Specific Suspension

```php
// Suspension rules by product
$productSuspensions = [
    'hosting' => [
        'suspend_days_after_due' => 14,
        'preserve_data_days' => 30,
        'suspend_actions' => [
            'disable_website',
            'disable_database',
            'disable_email',
            'show_suspension_page'
        ],
        'restore_includes' => [
            'restore_website',
            'restore_database',
            'restore_email'
        ]
    ],
    'domain' => [
        'suspend_days_after_due' => 0,    // Never suspend domains
        'registry_lock' => false,
        'hold_renewal' => true
    ],
    'reseller' => [
        'suspend_days_after_due' => 7,
        'suspend_child_accounts' => true,
        'preserve_data_days' => 14
    ],
    'dedicated_server' => [
        'suspend_days_after_due' => 3,
        'maintenance_mode' => true,
        'no_data_deletion' => true,
        'access_restricted' => true
    ]
];
```

## Suspension Process

### Suspension Workflow

```
1. Invoice Due Date Reached
2. Grace Period (7 days)
3. Payment Reminders Sent
4. [3 days before] Suspension Warning
5. Suspension Day Reached
6. Pre-Suspension Check
   - Extension active? -> Wait
   - Already suspended? -> Skip
7. Execute Suspension Actions
8. Send Suspension Notification
9. Update WHMCS Status
10. Log Suspension Event
```

### Suspension Actions

```php
// Actions performed on suspension
$suspensionActions = [
    // Hosting
    'suspend_hosting_account' => [
        'command' => 'cpanel_suspend',
        'wait_for_completion' => true,
        'timeout' => 60
    ],
    'disable_website' => [
        'method' => 'htaccess',
        'display_page' => 'suspended'
    ],
    'disable_database' => [
        'revoke_user' => true,
        'preserve_data' => true
    ],

    // Email
    'suspend_email_accounts' => [
        'disable_login' => true,
        'reject_incoming' => false,
        'preserve_mails' => true
    ],

    // Access
    'revoke_ftp_access' => true,
    'disable_ssh' => true,
    'update_dns' => [
        'add_suspension_record' => true,
        'redirect_to_suspension_page' => true
    ]
];
```

## Suspension Page

### Custom Suspension Page

```php
// Suspension page configuration
$suspensionPage = [
    'enabled' => true,
    'show_in_browser' => true,
    'show_in_app' => true,
    'page_type' => 'custom_html',

    // Page Content
    'page_title' => 'Account Suspended - Payment Required',
    'page_content' => 'Your service has been suspended due to non-payment.
                      Please make payment to restore service.',

    // Styling
    'template' => 'default_suspension',
    'logo_url' => '/assets/img/logo.png',

    // Contact
    'support_email' => 'support@example.com',
    'support_phone' => '+1-555-123-4567',
    'payment_url' => '/billing/pay/'
];
```

### Suspension Page Template

```html
<!DOCTYPE html>
<html>
<head>
    <title>Service Suspended</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding: 50px;
            background: #f5f5f5;
        }
        .suspended-container {
            background: white;
            padding: 40px;
            max-width: 600px;
            margin: 0 auto;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        .suspended-icon {
            font-size: 64px;
            color: #e74c3c;
        }
        .amount-due {
            font-size: 24px;
            color: #333;
            margin: 20px 0;
        }
        .pay-button {
            display: inline-block;
            background: #27ae60;
            color: white;
            padding: 15px 40px;
            text-decoration: none;
            border-radius: 5px;
            font-size: 18px;
        }
    </style>
</head>
<body>
    <div class="suspended-container">
        <div class="suspended-icon">&#9888;</div>
        <h1>Service Suspended</h1>
        <p>Your service has been suspended due to non-payment.</p>
        <p class="amount-due">Amount Due: $100.00</p>
        <a href="/billing/pay/{invoice_id}" class="pay-button">Pay Now & Restore</a>
        <p>Questions? Contact us at support@example.com</p>
    </div>
</body>
</html>
```

## API Reference

### Suspend Service

```http
POST /services/{service_id}/suspend
```

**Request Body:**

```json
{
  "reason" => "non_payment",
  "invoice_id" => "INV-12345",
  "notify_customer" => true,
  "preserve_data" => true
}
```

**Response:**

```json
{
  "success" => true,
  "service_id" => 67890,
  "suspended_at" => "2024-01-15T10:30:00Z",
  "suspension_reason" => "non_payment",
  "restore_available" => true,
  "restoration_time_hours" => 1
}
```

### Unsuspend Service

```http
POST /services/{service_id}/unsuspend
```

**Request Body:**

```json
{
  "payment_received" => true,
  "invoice_id" => "INV-12345",
  "restore_now" => true,
  "notify_customer" => true
}
```

### Get Suspension Status

```http
GET /services/{service_id}/suspension
```

**Response:**

```json
{
  "service_id" => 67890,
  "suspended" => true,
  "suspended_at" => "2024-01-15T10:30:00Z",
  "suspension_reason" => "non_payment",
  "invoice_id" => "INV-12345",
  "suspension_method" => "automated",
  "data_preserved" => true,
  "restore_available" => true,
  "unsuspend_command_sent" => false
}
```

## Restoration

### Auto Restoration

```php
// Auto-restore configuration
$autoRestore = [
    'enabled' => true,
    'trigger' => 'payment_received',
    'restore_delay_minutes' => 60,
    'check_payment_verification' => true,
    'payment_verification_wait' => 15,  // minutes

    // Restoration Steps
    'restore_website' => true,
    'restore_database' => true,
    'restore_email' => true,
    'restore_ftp' => true,
    'send_restore_notification' => true
];
```

### Manual Restoration

```http
POST /services/{service_id}/restore
```

**Request Body:**

```json
{
  "reason" => "Customer called and paid",
  "restored_by" => "admin@example.com",
  "notify_customer" => true
}
```

## Notifications

### Suspension Notifications

```php
// Suspension notification settings
$suspensionNotifications = [
    'before_suspension' => [
        'enabled' => true,
        'days_before' => 3,
        'template' => 'suspension_warning',
        'channels' => ['email', 'sms']
    ],
    'suspension_executed' => [
        'enabled' => true,
        'template' => 'service_suspended',
        'channels' => ['email']
    ],
    'restoration_complete' => [
        'enabled' => true,
        'template' => 'service_restored',
        'channels' => ['email']
    ],
    'admin_notification' => [
        'enabled' => true,
        'template' => 'admin_suspension_notice',
        'recipients' => ['billing@example.com', 'support@example.com']
    ]
];
```

## Exceptions

### Exempt Customers

```php
// Customers exempt from suspension
$suspensionExemptions = [
    'enterprise_customers' => [
        'enabled' => true,
        'groups' => ['enterprise'],
        'suspension_days' => 30,
        'require_approval' => true,
        'approval_required_from' => 'finance_manager'
    ],
    'good_standing' => [
        'enabled' => true,
        'payment_history_months' => 12,
        'on_time_percentage' => 98,
        'suspension_days' => 21
    ],
    'active_dispute' => [
        'enabled' => true,
        'auto_check' => true,
        'suspension_paused' => true
    ]
];
```

### Manual Suspension Override

```php
// Override automatic suspension
$manualOverride = [
    'enabled' => true,
    'require_reason' => true,
    'reason_options' => [
        'customer_contacted',
        'payment_arrangement',
        'billing_dispute',
        'service_issue',
        'other'
    ],
    'set_new_suspend_date' => true,
    'notify_on_override' => true
];
```

## Reporting

### Suspension Report

```http
GET /billing/reports/suspensions
```

**Query Parameters:**
- `period`: daily, weekly, monthly
- `date_from`: Start date
- `date_to`: End date

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_suspensions" => 50,
    "auto_suspensions" => 45,
    "manual_suspensions" => 5,
    "total_restorations" => 40,
    "auto_restored" => 35,
    "manual_restored" => 5,
    "still_suspended" => 10
  },
  "by_product" => [
    {"product" => "hosting", "suspensions" => 30, "restored" => 25},
    {"product" => "reseller", "suspensions" => 15, "restored" => 12},
    {"product" => "vps", "suspensions" => 5, "restored" => 3}
  ],
  "revenue_impact" => {
    "lost_revenue_days" => 500,
    "restored_revenue_days" => 450,
    "permanently_lost" => 50
  }
}
```

## Customer Portal

### Suspension Notice

**Client Area > Service Suspended**

```
+------------------------------------------------------------------+
|  SERVICE SUSPENDED                                               |
+------------------------------------------------------------------+
|                                                                  |
|  Your service has been suspended due to non-payment.            |
|                                                                  |
|  Service: Premium Hosting                                        |
|  Suspended At: January 15, 2024 at 10:30 AM                    |
|  Reason: Non-payment of Invoice INV-12345                      |
|                                                                  |
|  Amount Due: $100.00                                            |
|                                                                  |
|  +--------------------------------------------------------------+|
|  | WHAT HAPPENS NOW?                                           ||
|  |                                                             ||
|  | - Your website is temporarily offline                       ||
|  | - Email is temporarily unavailable                          ||
|  | - Your data is preserved for 30 days                        ||
|  |                                                             ||
|  | To restore service, please make payment below.              ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  [Pay Invoice Now - $100.00]                                   |
|                                                                  |
|  Need help? Contact support@example.com                        |
+------------------------------------------------------------------+
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Suspension not executing | Cron not running | Check automation |
| Data deleted | Wrong settings | Verify retention |
| Multiple suspensions | Script running twice | Check batch size |
| Restore failing | API timeout | Check server module |

### Debug Commands

```bash
# Check suspension status
whmcscli service suspension --service_id=67890

# Suspend manually
whmcscli service suspend --service_id=67890 --reason="manual"

# Unsuspend service
whmcscli service unsuspend --service_id=67890 --payment=received

# View suspended services
whmcscli service list --status=suspended
```

## Best Practices

1. **Communicate early** - Send warnings before suspension
2. **Preserve data** - Protect customer data during suspension
3. **Make payment easy** - Clear call-to-action
4. **Restore quickly** - Process payments promptly
5. **Document everything** - Log suspension reasons
6. **Review regularly** - Analyze suspension patterns

## See Also

- [Grace Period](./whmcs-grace-period.md)
- [Termination Policy](./whmcs-termination-policy.md)
- [Payment Terms](./whmcs-payment-terms.md)
