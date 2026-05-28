# WHMCS Overdue Handling Workflow

## Purpose

Automate the process of handling overdue invoices, from initial reminders through suspension and final termination, with configurable escalation paths.

## Prerequisites

- WHMCS with overdue invoice handling configured
- Cron job running for daily automation
- Email templates configured for overdue notifications
- Suspension and termination module configured
- Clear policies defined for overdue handling stages

## Workflow Steps

### Step 1: Configure Overdue Settings

Set up automation settings in WHMCS:

```php
// WHMCS Admin > System > Automation Settings
// Configure:
// - Overdue Invoice Reminder Days: 1, 3, 7, 14
// - Auto Suspension Days: 7
// - Auto Termination Days: 14

// Database configuration for advanced settings
INSERT INTO tblconfiguration (setting, value) VALUES 
('OverdueReminderDays', '1,3,7,14'),
('AutoSuspendDays', '7'),
('AutoTerminationDays', '14'),
('OverdueFeeEnabled', 'on'),
('OverdueFeeAmount', '5.00'),
('OverdueFeeInvoiceTemplate', 'Overdue Invoice Fee');
```

### Step 2: Create Overdue Invoice Hooks

Handle overdue invoice events:

```php
// File: /includes/hooks/overdue_handling.php

add_hook('InvoiceOverdue', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    $userId = $vars['userid'];
    $daysOverdue = $vars['days_overdue'];
    
    // Log the overdue event
    logActivity("Invoice #{$invoiceId} is {$daysOverdue} days overdue");
    
    // Get client details
    $client = Capsule::table('tblclients')
        ->where('id', $userId)
        ->first();
    
    // Track overdue history
    Capsule::table('mod_overdue_tracking')->insert([
        'invoice_id' => $invoiceId,
        'user_id' => $userId,
        'days_overdue' => $daysOverdue,
        'event_type' => 'reminder_sent',
        'created_at' => Capsule::raw('NOW()')
    ]);
    
    // Escalate based on days overdue
    if ($daysOverdue >= 3 && $daysOverdue < 7) {
        // First escalation - direct manager notification
        sendEscalationEmail('first_reminder', $client);
        
    } elseif ($daysOverdue >= 7 && $daysOverdue < 14) {
        // Second escalation - suspend services
        suspendOverdueServices($userId, $invoiceId);
        
    } elseif ($daysOverdue >= 14) {
        // Final escalation - terminate services
        terminateOverdueServices($userId, $invoiceId);
    }
});

function sendEscalationEmail($level, $client) {
    $templateMap = [
        'first_reminder' => 'InvoiceOverdueFirst',
        'second_reminder' => 'InvoiceOverdueSecond',
        'final_reminder' => 'InvoiceOverdueFinal'
    ];
    
    $template = $templateMap[$level] ?? 'InvoiceOverdueReminder';
    sendTemplatedEmail($template, $client->id, [
        'client_name' => $client->firstname . ' ' . $client->lastname
    ]);
}
```

### Step 3: Implement Service Suspension

Handle overdue service suspension:

```php
// File: /includes/hooks/overdue_suspension.php

function suspendOverdueServices($userId, $invoiceId) {
    // Get all active services for this client
    $services = Capsule::table('tblhosting')
        ->where('userid', $userId)
        ->where('domainstatus', 'Active')
        ->get();
    
    foreach ($services as $service) {
        // Get product module
        $product = Capsule::table('tblproducts')
            ->where('id', $service->packageid)
            ->first();
        
        if ($product->servertype) {
            // Run module suspend function
            $params = [
                'accountid' => $service->id,
                'username' => $service->username ?? '',
                'server' => getServerParams($service->server),
                'serverid' => $service->server
            ];
            
            $result = Capsule::moduleFactory($product->servertype)
                ->call('SuspendAccount', $params);
            
            if ($result === 'success') {
                Capsule::table('tblhosting')
                    ->where('id', $service->id)
                    ->update(['domainstatus' => 'Suspended']);
                
                logActivity("Service {$service->id} suspended due to overdue invoice #{$invoiceId}");
            }
        } else {
            // Non-module service - just update status
            Capsule::table('tblhosting')
                ->where('id', $service->id)
                ->update(['domainstatus' => 'Suspended']);
        }
        
        // Track suspension
        Capsule::table('mod_overdue_tracking')->insert([
            'invoice_id' => $invoiceId,
            'user_id' => $userId,
            'service_id' => $service->id,
            'event_type' => 'service_suspended',
            'created_at' => Capsule::raw('NOW()')
        ]);
    }
    
    // Send suspension notification to client
    $client = Capsule::table('tblclients')->where('id', $userId)->first();
    sendTemplatedEmail('ServicesSuspended', $client->id, [
        'invoice_id' => $invoiceId,
        'reason' => 'Overdue payment'
    ]);
}
```

### Step 4: Implement Service Termination

Handle overdue service termination:

```php
// File: /includes/hooks/overdue_termination.php

function terminateOverdueServices($userId, $invoiceId) {
    // Get all services (including suspended)
    $services = Capsule::table('tblhosting')
        ->where('userid', $userId)
        ->whereIn('domainstatus', ['Active', 'Suspended'])
        ->get();
    
    foreach ($services as $service) {
        // Get product module
        $product = Capsule::table('tblproducts')
            ->where('id', $service->packageid)
            ->first();
        
        if ($product->servertype) {
            // Run module terminate function
            $params = [
                'accountid' => $service->id,
                'username' => $service->username ?? '',
                'server' => getServerParams($service->server),
                'serverid' => $service->server
            ];
            
            $result = Capsule::moduleFactory($product->servertype)
                ->call('TerminateAccount', $params);
            
            if ($result === 'success') {
                Capsule::table('tblhosting')
                    ->where('id', $service->id)
                    ->update(['domainstatus' => 'Terminated']);
                
                logActivity("Service {$service->id} terminated due to overdue invoice #{$invoiceId}");
            }
        } else {
            Capsule::table('tblhosting')
                ->where('id', $service->id)
                ->update(['domainstatus' => 'Terminated']);
        }
        
        // Track termination
        Capsule::table('mod_overdue_tracking')->insert([
            'invoice_id' => $invoiceId,
            'user_id' => $userId,
            'service_id' => $service->id,
            'event_type' => 'service_terminated',
            'created_at' => Capsule::raw('NOW()')
        ]);
    }
    
    // Send termination notification
    $client = Capsule::table('tblclients')->where('id', $userId)->first();
    sendTemplatedEmail('ServicesTerminated', $client->id, [
        'invoice_id' => $invoiceId,
        'termination_date' => date('Y-m-d H:i:s')
    ]);
    
    // Mark client as high risk for future orders
    Capsule::table('tblclients')
        ->where('id', $userId)
        ->update([
            'overrideoverdueemails' => 1,
            'notes' => Capsule::raw("CONCAT(COALESCE(notes,''), '\n[" . date('Y-m-d') . "] Services terminated due to overdue invoice #{$invoiceId}')")
        ]);
}
```

### Step 5: Implement Overdue Fee Application

Apply late fees for overdue invoices:

```php
// File: /includes/hooks/overdue_fees.php

add_hook('DailyCronJob', 1, function($vars) {
    // Get settings
    $feeEnabled = Capsule::table('tblconfiguration')
        ->where('setting', 'OverdueFeeEnabled')
        ->value('value');
    
    if ($feeEnabled !== 'on') {
        return;
    }
    
    $feeAmount = (float) Capsule::table('tblconfiguration')
        ->where('setting', 'OverdueFeeAmount')
        ->value('value') ?? 5.00;
    
    // Find invoices overdue by more than specified days
    $overdueDays = 7; // Apply fee after 7 days
    $cutoffDate = date('Y-m-d', strtotime('-' . $overdueDays . ' days'));
    
    $overdueInvoices = Capsule::table('tblinvoices')
        ->where('status', 'Overdue')
        ->where('duedate', '<', $cutoffDate)
        ->where('id', 'NOT IN', function($query) {
            // Exclude invoices that already have overdue fee
            $query->select('invoiceid')
                ->from('tblinvoiceitems')
                ->where('type', 'OverdueFee');
        })
        ->get();
    
    foreach ($overdueInvoices as $invoice) {
        // Check if fee was already applied recently
        $recentFee = Capsule::table('mod_overdue_fees')
            ->where('invoice_id', $invoice->id)
            ->where('created_at', '>', date('Y-m-d', strtotime('-30 days')))
            ->first();
        
        if ($recentFee) {
            continue;
        }
        
        // Create fee invoice
        applyOverdueFee($invoice, $feeAmount);
    }
});

function applyOverdueFee($invoice, $feeAmount) {
    // Create or add to existing fee invoice
    $feeInvoiceId = Capsule::table('tblinvoices')->insertGetId([
        'userid' => $invoice->userid,
        'invoicenum' => '',
        'date' => date('Y-m-d'),
        'duedate' => date('Y-m-d'),
        'datepaid' => null,
        'status' => 'Overdue',
        'subtotal' => $feeAmount,
        'tax' => 0,
        'tax2' => 0,
        'total' => $feeAmount,
        'balance' => $feeAmount,
        'paymentmethod' => $invoice->paymentmethod,
        'notes' => "Overdue fee for Invoice #{$invoice->id}",
        'created_at' => Capsule::raw('NOW()')
    ]);
    
    // Add line item
    Capsule::table('tblinvoiceitems')->insert([
        'invoiceid' => $feeInvoiceId,
        'userid' => $invoice->userid,
        'type' => 'OverdueFee',
        'description' => 'Late payment fee',
        'amount' => $feeAmount
    ]);
    
    // Track the fee
    Capsule::table('mod_overdue_fees')->insert([
        'invoice_id' => $invoice->id,
        'fee_invoice_id' => $feeInvoiceId,
        'amount' => $feeAmount,
        'created_at' => Capsule::raw('NOW()')
    ]);
    
    logActivity("Applied overdue fee of {$feeAmount} to client #{$invoice->userid}");
}
```

### Step 6: Overdue Recovery Hook

Attempt to recover overdue invoices:

```php
// File: /includes/hooks/overdue_recovery.php

add_hook('DailyCronJob', 1, function($vars) {
    // Check for accounts with multiple overdue invoices
    $chronicOverdue = Capsule::table('tblclients')
        ->join('tblinvoices', 'tblclients.id', '=', 'tblinvoices.userid')
        ->where('tblinvoices.status', 'Overdue')
        ->groupBy('tblclients.id')
        ->havingRaw('COUNT(*) >= 3')
        ->select('tblclients.id', 'tblclients.email')
        ->get();
    
    foreach ($chronicOverdue as $client) {
        // Send final notice
        sendTemplatedEmail('FinalOverdueNotice', $client->id, [
            'total_overdue' => getClientOverdueTotal($client->id),
            'invoice_count' => getClientOverdueCount($client->id)
        ]);
        
        // Flag for manual review
        Capsule::table('mod_overdue_tracking')->insert([
            'user_id' => $client->id,
            'event_type' => 'chronic_overdue_flagged',
            'created_at' => Capsule::raw('NOW()')
        ]);
    }
    
    // Report summary to admin
    $report = [
        'date' => date('Y-m-d'),
        'new_overdue' => getNewOverdueCount(),
        'services_suspended' => getSuspensionCount(),
        'services_terminated' => getTerminationCount(),
        'fees_applied' => getOverdueFeeCount()
    ];
    
    if ($report['services_terminated'] > 0) {
        sendAdminNotification(
            'email',
            'Overdue Handling Report',
            json_encode($report, JSON_PRETTY_PRINT)
        );
    }
});
```

## Verification Checklist

- [ ] Overdue settings configured in WHMCS automation
- [ ] Email templates created for all reminder stages
- [ ] Suspension logic tested with test accounts
- [ ] Termination process verified
- [ ] Overdue fees applied correctly
- [ ] Client notifications sent at each stage
- [ ] Admin reports generated correctly
- [ ] Tracking table capturing all events
- [ ] Cron job running on schedule
- [ ] Manual intervention working for exceptions

## Related Skills and Documentation

- [WHMCS Invoice Automation](whmcs-invoice-automation-workflow.md)
- [WHMCS Subscription Workflow](whmcs-subscription-workflow.md)
- [WHMCS Payment Processing](whmcs-payment-processing-workflow.md)
- WHMCS Documentation: Invoice Overdue Handling
- WHMCS Documentation: Service Suspension

## Notes

- Clearly communicate policies to clients upfront
- Review overdue handling metrics monthly
- Adjust timelines based on business requirements
- Consider payment plan options for chronic late payers
- Document all exceptions for audit purposes
- Balance collection efforts with customer retention