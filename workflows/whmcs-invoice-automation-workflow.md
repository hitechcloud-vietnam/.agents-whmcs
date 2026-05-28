# WHMCS Invoice Automation Workflow

## Purpose

Automate invoice generation, sending, and tracking in WHMCS to reduce manual billing operations and ensure timely client notifications.

## Prerequisites

- WHMCS installation with administrative access
- Cron job configured for automated tasks
- Email templates configured for invoice notifications
- Payment gateway configured and active
- Client accounts with billing information complete

## Workflow Steps

### Step 1: Configure Invoice Automation Settings

Set up automatic invoice generation in WHMCS:

```php
// Configuration in WHMCS Admin > Setup > Automation Settings
// Set "Generate Invoices" to enabled
// Configure "Days Before Due Date" for invoice generation timing

// Direct database configuration for advanced settings
INSERT INTO tblconfiguration (setting, value) VALUES 
('AutoGenerateInvoices', 'on'),
('InvoiceGenerationDaysAdvance', '3'),
('AutoSuspendOnOverdue', 'on'),
('OverdueSuspensionDays', '14');
```

### Step 2: Set Up Invoice Templates

Customize invoice email templates for automated sends:

```php
// File: /resources/views/invoice-template.html
// WHMCS Admin > System > Email Templates > Invoice Created

// Use template variables for dynamic content:
/*
Subject: Invoice #{$invoice_num} Generated - Due: {$due_date}

Dear {$client_name},

Your invoice #{$invoice_num} has been generated.

Invoice Date: {$invoice_date}
Due Date: {$due_date}
Total Due: {$total}

View and pay online: {$invoice_num}
{$invoice_url}

Thank you for your business!
*/

// For reminder emails, configure "Invoice Reminder" template
```

### Step 3: Configure Cron Job for Automation

Ensure the WHMCS cron job runs for invoice automation:

```bash
# Add to crontab (run as web server user)
# /etc/crontab or via WHMCS Admin > Configuration > Systemcron

# WHMCS recommends every 5 minutes for optimal automation
*/5 * * * * php -q /var/www/whmcs/crons/cron.php

# Alternative: run specific automation tasks
# Invoice generation
php -q /var/www/whmcs/crons/cron.php doInvoiceGeneration

# Overdue invoice handling
php -q /var/www/whmcs/crons/cron.php doOverdueInvoiceNotification
```

### Step 4: Create Invoice Generation Hook

Custom hook for specialized invoice handling:

```php
// File: /includes/hooks/invoice_automation.php

add_hook('InvoiceCreated', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    
    // Log the automated invoice creation
    logActivity("Auto-generated invoice #{$invoiceId} created");
    
    // Update custom tracking table
    Capsule::table('mod_invoice_tracking')->insert([
        'invoice_id' => $invoiceId,
        'created_via' => 'automation',
        'created_at' => Capsule::raw('NOW()')
    ]);
    
    // Send additional notification via custom module
    if (file_exists(__DIR__ . '/../modules/notifications/customwebhook.php')) {
        $notification = new \WHMCS\Notification\Webhook();
        $notification->send([
            'event' => 'invoice_created',
            'invoice_id' => $invoiceId,
            'amount' => $vars['total'],
            'client_id' => $vars['userid']
        ]);
    }
});

add_hook('InvoicePaid', 1, function($vars) {
    // Track paid invoices for analytics
    Capsule::table('mod_invoice_tracking')
        ->where('invoice_id', $vars['invoiceid'])
        ->update([
            'paid_at' => Capsule::raw('NOW()'),
            'payment_method' => $vars['paymentmethod']
        ]);
});
```

### Step 5: Implement Invoice Reminder System

Configure automated reminder notifications:

```php
// Configure in WHMCS Admin > System > Automation Settings
// Set reminder emails for:
// - 3 days before due date
// - On due date
// - 1 day after due date
// - 7 days overdue
// - 14 days overdue

// Custom reminder hook for additional actions
add_hook('InvoiceOverdue', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    $clientId = $vars['userid'];
    
    // Get client details for escalation
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    // Check if client has multiple overdue invoices
    $overdueCount = Capsule::table('tblinvoices')
        ->where('userid', $clientId)
        ->where('status', 'Overdue')
        ->count();
    
    if ($overdueCount > 2) {
        // Escalate to billing department
        logTicket(
            'Billing Escalation',
            "Client {$client->firstname} {$client->lastname} has {$overdueCount} overdue invoices.",
            'Billing',
            'High'
        );
    }
});
```

### Step 6: Create Invoice Report Hook

Track automation metrics:

```php
// File: /includes/hooks/invoice_reporting.php

add_hook('DailyCronJob', 1, function($vars) {
    // Generate daily invoice automation report
    $stats = [
        'generated' => Capsule::table('tblinvoices')
            ->whereDate('date', '=', date('Y-m-d'))
            ->where('status', '!=', 'Cancelled')
            ->count(),
        'paid' => Capsule::table('tblinvoices')
            ->whereDate('datepaid', '=', date('Y-m-d'))
            ->count(),
        'overdue' => Capsule::table('tblinvoices')
            ->where('status', 'Overdue')
            ->count(),
        'total_overdue_amount' => Capsule::table('tblinvoices')
            ->where('status', 'Overdue')
            ->sum('total')
    ];
    
    // Log stats for monitoring
    $report = "Invoice Automation Report - " . date('Y-m-d H:i:s') . "\n";
    $report .= "Generated: {$stats['generated']}\n";
    $report .= "Paid: {$stats['paid']}\n";
    $report .= "Overdue: {$stats['overdue']} ({$stats['total_overdue_amount']})\n";
    
    file_put_contents(
        __DIR__ . '/../storage/logs/invoice_automation.log',
        $report,
        FILE_APPEND
    );
    
    // Send report to admin if overdue exceeds threshold
    if ($stats['overdue'] > 50 || $stats['total_overdue_amount'] > 10000) {
        sendAdminNotification(
            'email',
            'Invoice Alert: High Overdue Rate',
            $report
        );
    }
});
```

## Verification Checklist

- [ ] Cron job is running every 5 minutes
- [ ] Invoice generation settings configured correctly
- [ ] Email templates created and tested
- [ ] Invoice reminders are set up in automation settings
- [ ] Hooks are properly registered (check /includes/hooks/)
- [ ] Test invoice generation manually
- [ ] Verify email notifications are received
- [ ] Check automation logs for successful runs
- [ ] Monitor overdue invoice rate
- [ ] Review automation report for accuracy

## Related Skills and Documentation

- [WHMCS Webhook Automation](whmcs-webhook-automation-workflow.md)
- [WHMCS Payment Gateway Setup](whmcs-payment-gateway-setup-workflow.md)
- WHMCS Documentation: Invoice Automation
- WHMCS Documentation: Email Templates
- WHMCS Documentation: Cron Automation

## Notes

- Always test invoice automation in staging before production
- Monitor email delivery rates and bounced emails
- Keep invoice templates updated with company branding
- Review overdue invoice settings monthly
- Consider adding late payment fees for repeat offenders