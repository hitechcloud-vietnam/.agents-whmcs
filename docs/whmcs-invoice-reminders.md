# WHMCS Invoice Reminders

## Overview

Invoice reminders are automated notifications sent to clients regarding their unpaid invoices. WHMCS supports configurable reminder schedules, multiple reminder types, and customizable email templates.

## Reminder Configuration

### Accessing Reminder Settings

**Configuration > Billing > Invoice Settings > Reminders**

### Basic Settings

```php
[
    'reminders_enabled' => true,
    'days_before_due' => 7,           // Send reminder X days before due
    'days_after_due' => 3,             // Send first overdue reminder
    'include_payment_link' => true,
    'attach_invoice' => true
]
```

## Reminder Schedule

### Default Schedule

| Reminder | Day | Description |
|----------|-----|-------------|
| Pre-Due | -7 | 7 days before due date |
| Due Date | 0 | On due date |
| First Overdue | 3 | 3 days after due date |
| Second Overdue | 7 | 1 week after due date |
| Third Overdue | 14 | 2 weeks after due date |
| Final Notice | 21 | 3 weeks after due date |

### Custom Schedule

```php
// Configure custom reminder timeline
[
    'reminders' => [
        ['day' => -7, 'template' => 'reminder_7_days', 'active' => true],
        ['day' => -3, 'template' => 'reminder_3_days', 'active' => true],
        ['day' => 0, 'template' => 'due_date_reminder', 'active' => true],
        ['day' => 3, 'template' => 'overdue_3_days', 'active' => true],
        ['day' => 7, 'template' => 'overdue_1_week', 'active' => true],
        ['day' => 14, 'template' => 'overdue_2_weeks', 'active' => true],
        ['day' => 21, 'template' => 'final_notice', 'active' => true]
    ]
]
```

### Per-Product Reminders

```php
// Different schedule for specific products
[
    'product_id' => 1,
    'custom_reminders' => true,
    'reminders' => [
        ['day' => -14, 'template' => 'hosting_reminder'],
        ['day' => -7, 'template' => 'hosting_reminder'],
        ['day' => 0, 'template' => 'hosting_due']
    ]
]
```

## Email Templates

### Template Configuration

**Configuration > Emails > Invoice Reminders**

### Reminder Variables

```smarty
<!-- Available template variables -->
{$invoice_number}           <!-- INV-2024-0123 -->
{$invoice_amount}           <!-- $100.00 -->
{$amount_due}                <!-- $100.00 -->
{$balance}                   <!-- $100.00 -->
{$due_date}                  <!-- May 15, 2024 -->
{$days_until_due}            <!-- -7 (if pre-due) -->
{$days_overdue}              <!-- 3 (if overdue) -->
{$client_name}               <!-- John Doe -->
{$client_first_name}         <!-- John -->
{$payment_link}              <!-- https://... -->
{$invoice_url}              <!-- https://... -->
{$company_name}              <!-- Your Company -->
{$company_logo}              <!-- https://... -->
```

### Sample Templates

```html
<!-- Pre-Due Reminder Template -->
Subject: Upcoming Invoice Due - {$invoice_number}

Dear {$client_first_name},

This is a friendly reminder that invoice {$invoice_number} for {$amount_due} 
is due in {$days_until_due} days on {$due_date}.

You can view and pay your invoice here:
{$payment_link}

Thank you for your business!

{$company_name}
```

```html
<!-- Overdue Reminder Template -->
Subject: Payment Overdue - Invoice {$invoice_number}

Dear {$client_first_name},

Your invoice {$invoice_number} for {$amount_due} was due on {$due_date} 
and is now {$days_overdue} days overdue.

Please make payment as soon as possible to avoid service interruption.

Pay Now: {$payment_link}

If you have already sent payment, please disregard this notice.

{$company_name}
```

## Reminder Content

### Include Information

```php
// What to include in reminders
[
    'include_invoice_details' => true,
    'include_line_items' => true,
    'include_balance' => true,
    'include_due_date' => true,
    'include_late_fee_notice' => true,
    'attach_pdf' => true,
    'attach_invoice_pdf' => true
]
```

### Payment Details

```php
// Always include payment options
[
    'payment_methods' => 'all',
    'include_credit_option' => true,
    'include_payment_button' => true,
    'payment_button_text' => 'Pay Now'
]
```

## Reminder Conditions

### Amount-Based

```php
// Only send for amounts above threshold
[
    'minimum_amount' => 1.00,
    'maximum_amount' => null
]
```

### Client-Based

```php
// Exclude certain clients
[
    'exclude_client_groups' => ['premium', 'vip'],
    'exclude_clients' => [123, 456]
]
```

### Invoice-Based

```php
// Exclude certain invoices
[
    'exclude_invoice_types' => ['proforma', 'quote'],
    'exclude_due_in_future_days' => 30
]
```

## Manual Reminders

### Send Individual Reminder

**Admin: Billing > Invoices > Select Invoice > Send Reminder**

```php
// Manual reminder options
[
    'invoice_id' => 5678,
    'template' => 'custom_reminder',
    'custom_message' => 'We noticed you may need assistance...',
    'attach_invoice' => true,
    'cc_admin' => false
]
```

### Bulk Reminders

```php
// Send reminders for multiple invoices
[
    'action' => 'send_bulk_reminders',
    'filter' => [
        'status' => 'Overdue',
        'days_overdue' => '>=7',
        'amount_min' => 10.00
    ],
    'template' => 'overdue_reminder',
    'max_to_send' => 100
]
```

## Reminder Suppression

### Auto-Suppress

```php
// Don't resend if already paid
[
    'suppress_if_paid' => true,
    'suppress_if_awaiting_payment' => true,
    'suppress_if_credit_applied' => true
]
```

### Manual Suppress

```php
// Skip specific reminder
[
    'invoice_id' => 5678,
    'skip_next_reminder' => true,
    'skip_until_date' => '2024-05-20',
    'reason' => 'Client contact established'
]
```

## Reminder Logging

### Track Reminders Sent

```php
// Reminder history
[
    'invoice_id' => 5678,
    'reminders' => [
        ['date' => '2024-05-01', 'type' => 'pre_due', 'sent_to' => 'client@email.com'],
        ['date' => '2024-05-04', 'type' => 'due_date', 'sent_to' => 'client@email.com'],
        ['date' => '2024-05-07', 'type' => 'first_overdue', 'sent_to' => 'client@email.com'],
        ['date' => '2024-05-10', 'type' => 'second_overdue', 'sent_to' => 'client@email.com']
    ]
]
```

## Integration with Dunning

### Dunning Integration

```php
// Reminders as part of dunning
[
    'dunning_enabled' => true,
    'reminders' => [
        ['day' => 0, 'action' => 'reminder', 'dunning_stage' => 1],
        ['day' => 3, 'action' => 'reminder', 'dunning_stage' => 2],
        ['day' => 7, 'action' => 'warning', 'dunning_stage' => 3],
        ['day' => 14, 'action' => 'suspend_warning', 'dunning_stage' => 4]
    ]
]
```

## Cron Configuration

### Automated Reminders

```bash
# Cron processes reminders
* * * * * php -q /whmcs/crons/autocron.php
```

### Reminder Processing

```php
// Cron runs daily
// 1. Check invoices for due reminders
// 2. Apply any conditions
// 3. Send reminder emails
// 4. Log each reminder sent
// 5. Update next reminder date
```

## API Functions

```php
// Send reminder
$params = [
    'invoiceid' => 5678,
    'template' => 'overdue_reminder'
];
$result = localAPI('SendInvoiceReminder', $params);

// Get reminder history
$params = [
    'invoiceid' => 5678
];
$result = localAPI('GetInvoiceReminders', $params);

// Update reminder settings
$params = [
    'reminders' => [
        ['day' => -7, 'enabled' => true],
        ['day' => 0, 'enabled' => true],
        ['day' => 3, 'enabled' => true]
    ]
];
$result = localAPI('UpdateReminderSettings', $params);
```

## Hooks

```php
// Hook: PreInvoiceReminder
add_hook('PreInvoiceReminder', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['reminder_day']
    // $vars['template']
    
    // Modify or prevent reminder
    return ['send' => true, 'template' => $vars['template']];
});

// Hook: InvoiceReminderSent
add_hook('InvoiceReminderSent', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['reminder_type']
    // $vars['email_to']
    // $vars['sent_at']
});
```

## Best Practices

1. **Start early**: Send reminders before due date
2. **Be clear**: Include amount and due date prominently
3. **Make it easy**: Include payment link
4. **Escalate tone**: Increase urgency with each reminder
5. **Track results**: Monitor which reminders work best

## Related Documentation

- [Invoice Generation](./whmcs-invoice-generation.md)
- [Dunning Settings](./whmcs-dunning-settings.md)
- [Late Fees](./whmcs-late-fees.md)
- [Email Templates](./whmcs-email-templates.md)