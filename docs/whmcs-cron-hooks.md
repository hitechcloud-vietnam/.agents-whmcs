# WHMCS Cron Hooks

## Overview

Cron hooks allow you to execute custom code during WHMCS scheduled tasks, enabling automation of maintenance, billing, and notification processes.

## Available Cron Hooks

### Daily Cron Hook

```php
<?php
// Runs daily with the WHMCS cron job
add_hook('DailyCronJob', 1, function(array $vars) {
    $date = $vars['date'];
    
    // Process overdue invoices
    processOverdueInvoices();
    
    // Send expiration reminders
    sendExpirationReminders();
    
    // Suspend overdue services
    suspendOverdueServices();
    
    // Generate usage reports
    generateUsageReports();
    
    // Sync with external systems
    syncExternalData();
    
    // Clean up old data
    cleanupOldData();
    
    return ['success' => true, 'processed_at' => date('Y-m-d H:i:s')];
});

function processOverdueInvoices(): void
{
    $overdueInvoices = Capsule::table('tblinvoices')
        ->where('status', 'Unpaid')
        ->where('duedate', '<', date('Y-m-d'))
        ->get();
    
    foreach ($overdueInvoices as $invoice) {
        $daysOverdue = (strtotime(date('Y-m-d')) - strtotime($invoice->duedate)) / 86400;
        
        // Escalate based on days overdue
        if ($daysOverdue >= 7) {
            escalateInvoice($invoice->id, $daysOverdue);
        }
        
        if ($daysOverdue >= 14) {
            suspendService($invoice->id);
        }
    }
}
```

### Hourly Cron Hook

```php
<?php
// Runs every hour
add_hook('HourlyCronJob', 1, function(array $vars) {
    // Process pending queue items
    processPendingQueue();
    
    // Check service health
    checkServiceHealth();
    
    // Process pending emails
    processPendingEmails();
    
    return ['success' => true];
});
```

### Weekly Cron Hook

```php
<?php
// Runs weekly
add_hook('WeeklyCronJob', 1, function(array $vars) {
    // Generate weekly reports
    generateWeeklyReports();
    
    // Cleanup old logs
    cleanupOldLogs();
    
    // Backup critical data
    performWeeklyBackup();
    
    // Review flagged accounts
    reviewFlaggedAccounts();
    
    return ['success' => true];
});
```

### Monthly Cron Hook

```php
<?php
// Runs monthly
add_hook('MonthlyCronJob', 1, function(array $vars) {
    // Generate monthly invoices
    generateMonthlyInvoices();
    
    // Calculate monthly reports
    calculateMonthlyReports();
    
    // Review and adjust pricing
    reviewPricing();
    
    // Archive old data
    archiveOldData();
    
    return ['success' => true];
});
```

## Invoice Processing Hooks

```php
<?php
// Runs during invoice creation cron
add_hook('InvoiceCreationCron', 1, function(array $vars) {
    $invoicesCreated = 0;
    
    // Get services due for billing
    $dueServices = getServicesDueForBilling();
    
    foreach ($dueServices as $service) {
        // Check for custom billing rules
        if (hasCustomBilling($service->id)) {
            $amount = calculateCustomAmount($service);
        } else {
            $amount = $service->amount;
        }
        
        // Only create invoice if amount > 0
        if ($amount > 0) {
            createInvoice($service, $amount);
            $invoicesCreated++;
        }
    }
    
    return ['success' => true, 'invoices_created' => $invoicesCreated];
});

function getServicesDueForBilling(): array
{
    return Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->where('nextduedate', date('Y-m-d'))
        ->where('billingcycle', 'Monthly')
        ->get();
}
```

### Late Fee Hook

```php
<?php
// Apply late fees to overdue invoices
add_hook('LateFeeCron', 1, function(array $vars) {
    $invoices = Capsule::table('tblinvoices')
        ->where('status', 'Unpaid')
        ->where('duedate', '<', date('Y-m-d', strtotime('-7 days')))
        ->get();
    
    foreach ($invoices as $invoice) {
        $daysLate = (strtotime(date('Y-m-d')) - strtotime($invoice->duedate)) / 86400;
        
        // Apply late fee once
        $existingLateFee = Capsule::table('tblinvoiceitems')
            ->where('invoice_id', $invoice->id)
            ->where('type', 'latefee')
            ->first();
        
        if (!$existingLateFee && $daysLate >= 7) {
            $lateFee = calculateLateFee($invoice->total);
            addInvoiceItem($invoice->id, 'Late Fee', $lateFee);
        }
    }
    
    return ['success' => true];
});
```

## Suspension/Termination Hooks

```php
<?php
// Runs during service suspension cron
add_hook('ServiceSuspensionCron', 1, function(array $vars) {
    // Find services overdue for suspension
    $servicesToSuspend = Capsule::table('tblhosting')
        ->join('tblinvoices', 'tblhosting.userid', '=', 'tblinvoices.userid')
        ->where('tblhosting.domainstatus', 'Active')
        ->where('tblinvoices.status', 'Unpaid')
        ->where('tblinvoices.duedate', '<', date('Y-m-d', strtotime('-7 days')))
        ->select('tblhosting.*')
        ->groupBy('tblhosting.id')
        ->get();
    
    foreach ($servicesToSuspend as $service) {
        $module = new ServerModule($service->servertype);
        $result = $module->SuspendAccount($service->id);
        
        if ($result === 'success') {
            Capsule::table('tblhosting')
                ->where('id', $service->id)
                ->update(['domainstatus' => 'Suspended']);
            
            logActivity("Auto-suspended service #{$service->id}");
        }
    }
    
    return ['success' => true, 'suspended_count' => count($servicesToSuspend)];
});
```

### Termination Cron

```php
<?php
// Runs during service termination cron
add_hook('ServiceTerminationCron', 1, function(array $vars) {
    // Find terminated services ready for deletion
    $servicesToDelete = Capsule::table('tblhosting')
        ->where('domainstatus', 'Terminated')
        ->where('termination_date', '<', date('Y-m-d', strtotime('-30 days')))
        ->get();
    
    foreach ($servicesToDelete as $service) {
        // Only delete if no pending invoices
        $pendingInvoices = Capsule::table('tblinvoices')
            ->where('userid', $service->userid)
            ->whereIn('status', ['Unpaid', 'Paid'])
            ->count();
        
        if ($pendingInvoices === 0) {
            deleteServiceData($service->id);
        }
    }
    
    return ['success' => true];
});
```

## Domain Hooks

```php
<?php
// Domain sync cron
add_hook('DomainSyncCron', 1, function(array $vars) {
    $domains = Capsule::table('tbldomains')
        ->where('status', 'Active')
        ->get();
    
    foreach ($domains as $domain) {
        $registrar = new RegistrarModule($domain->registrar);
        $sync = $registrar->Sync($domain->id);
        
        if ($sync['expiry'] !== $domain->nextduedate) {
            Capsule::table('tbldomains')
                ->where('id', $domain->id)
                ->update(['nextduedate' => $sync['expiry']]);
        }
        
        // Check for expiring domains
        if ($sync['status'] === 'Expired') {
            markDomainExpired($domain->id);
        }
    }
    
    return ['success' => true, 'synced_count' => count($domains)];
});
```

## Usage Statistics Hook

```php
<?php
// Usage statistics collection
add_hook('UsageStatsCron', 1, function(array $vars) {
    $services = Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->get();
    
    foreach ($services as $service) {
        $stats = collectUsageStats($service);
        
        Capsule::table('mod_service_usage')->insert([
            'service_id' => $service->id,
            'disk_usage' => $stats['disk'],
            'bandwidth_usage' => $stats['bandwidth'],
            'record_date' => date('Y-m-d'),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        // Check limits
        checkUsageLimits($service, $stats);
    }
    
    return ['success' => true];
});
```

## Comprehensive Cron Handler

```php
<?php
class CronHookHandler {
    
    public function register(): void
    {
        add_hook('DailyCronJob', 1, [$this, 'handleDaily']);
        add_hook('HourlyCronJob', 1, [$this, 'handleHourly']);
        add_hook('WeeklyCronJob', 1, [$this, 'handleWeekly']);
        add_hook('MonthlyCronJob', 1, [$this, 'handleMonthly']);
        add_hook('InvoiceCreationCron', 1, [$this, 'handleInvoiceCreation']);
        add_hook('ServiceSuspensionCron', 1, [$this, 'handleSuspensions']);
    }
    
    public function handleDaily(array $vars): array
    {
        $this->processOverdueInvoices();
        $this->sendReminders();
        $this->syncExternalData();
        $this->logCronRun('daily', $vars);
        return ['success' => true];
    }
    
    public function handleHourly(array $vars): array
    {
        $this->processQueues();
        $this->checkHealth();
        return ['success' => true];
    }
    
    public function handleWeekly(array $vars): array
    {
        $this->generateReports();
        $this->cleanupLogs();
        $this->performBackups();
        return ['success' => true];
    }
    
    public function handleMonthly(array $vars): array
    {
        $this->generateMonthlyInvoices();
        $this->archiveData();
        return ['success' => true];
    }
    
    public function handleInvoiceCreation(array $vars): array
    {
        $count = $this->createInvoices();
        return ['success' => true, 'invoices_created' => $count];
    }
    
    public function handleSuspensions(array $vars): array
    {
        $count = $this->suspendOverdueServices();
        return ['success' => true, 'suspended_count' => $count];
    }
    
    private function processOverdueInvoices(): void
    {
        // Process overdue invoices
    }
    
    private function sendReminders(): void
    {
        // Send reminders
    }
    
    private function syncExternalData(): void
    {
        // Sync external data
    }
    
    private function logCronRun(string $type, array $vars): void
    {
        Capsule::table('mod_cron_log')->insert([
            'cron_type' => $type,
            'run_at' => date('Y-m-d H:i:s'),
            'result' => json_encode($vars),
        ]);
    }
    
    private function processQueues(): void
    {
        // Process queues
    }
    
    private function checkHealth(): void
    {
        // Check health
    }
    
    private function generateReports(): void
    {
        // Generate reports
    }
    
    private function cleanupLogs(): void
    {
        // Cleanup logs
    }
    
    private function performBackups(): void
    {
        // Perform backups
    }
    
    private function createInvoices(): int
    {
        return 0;
    }
    
    private function suspendOverdueServices(): int
    {
        return 0;
    }
    
    private function generateMonthlyInvoices(): void
    {
        // Generate monthly invoices
    }
    
    private function archiveData(): void
    {
        // Archive data
    }
}

$handler = new CronHookHandler();
$handler->register();
```

## Best Practices

1. **Keep hooks fast** - Queue heavy operations for later
2. **Use transactions** - Ensure data consistency
3. **Log execution** - Track cron performance
4. **Handle errors gracefully** - Use try-catch and continue on failure
5. **Respect time limits** - Avoid long-running hooks

## Related Documentation

- [WHMCS Ticket Hooks](/docs/whmcs-ticket-hooks.md)
- [WHMCS Client Hooks](/docs/whmcs-client-hooks.md)