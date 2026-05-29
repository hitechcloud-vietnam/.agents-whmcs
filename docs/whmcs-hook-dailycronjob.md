# WHMCS DailyCronJob Hook Reference

## Overview

The `DailyCronJob` hook fires during WHMCS daily cron execution. This hook runs once per day and is ideal for batch processing, reporting, and automated maintenance tasks.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `cronlastrun` | string | Previous cron run timestamp |
| `crontnextrun` | string | Next cron run timestamp |
| `croncurrentrun` | string | Current cron run timestamp |

## Example Implementation

```php
<?php
add_hook('DailyCronJob', 1, function(array $params) {
    // Log cron execution
    logActivity("Daily cron started at " . date('Y-m-d H:i:s'));
    
    // Run daily maintenance tasks
    cleanupOldData();
    
    return $params;
});
```

## Billing and Suspension

```php
<?php
add_hook('DailyCronJob', 1, function(array $params) {
    // 1. Check for overdue invoices and suspend services
    $overdueServices = full_query("
        SELECT h.id, h.domain, h.userid 
        FROM tblhosting h
        INNER JOIN tblinvoices i ON i.userid = h.userid
        INNER JOIN tblinvoiceitems ii ON ii.invoiceid = i.id AND ii.relid = h.id AND ii.type = 'Hosting'
        WHERE i.status = 'Overdue'
        AND h.domainstatus = 'Active'
        GROUP BY h.id
    ");
    
    while ($service = mysql_fetch_array($overdueServices)) {
        // Check if overdue for more than grace period
        if (hasOverdueGracePeriodExpired($service['id'])) {
            $module = new \WHMCS\Module\Server();
            if ($module->load($service['id'])) {
                $module->call('Suspend');
            }
            logActivity("Auto-suspended service {$service['id']} for overdue invoice");
        }
    }
    
    // 2. Send payment reminders
    $reminderDays = [7, 3, 1]; // Days before due date
    foreach ($reminderDays as $days) {
        sendPaymentReminders($days);
    }
    
    // 3. Process auto-renewal notices for domains
    processDomainRenewalNotices();
    
    // 4. Generate monthly reports
    generateMonthlyReports();
    
    return $params;
});
```

## Data Cleanup and Maintenance

```php
<?php
add_hook('DailyCronJob', 1, function(array $params) {
    // 1. Clean up expired sessions
    full_query("DELETE FROM tblsessions WHERE expires < UNIX_TIMESTAMP()");
    
    // 2. Clean up old activity logs (retention policy)
    $retentionDays = 90;
    $cutoffDate = date('Y-m-d H:i:s', strtotime("-{$retentionDays} days"));
    full_query("DELETE FROM tblactivitylog WHERE created_at < '{$cutoffDate}'");
    
    // 3. Archive old tickets
    $archiveDays = 180;
    $archiveDate = date('Y-m-d', strtotime("-{$archiveDays} days"));
    full_query("
        INSERT INTO tbltickets_archive 
        SELECT * FROM tbltickets 
        WHERE status IN ('Closed', 'Resolved') 
        AND lastreply < '{$archiveDate}'
    ");
    
    // 4. Remove orphaned cache files
    cleanOrphanedCacheFiles();
    
    // 5. Optimize database tables
    foreach (getDatabaseTables() as $table) {
        full_query("OPTIMIZE TABLE {$table}");
    }
    
    return $params;
});
```

## Reporting and Analytics

```php
<?php
add_hook('DailyCronJob', 1, function(array $params) {
    // 1. Generate daily statistics
    $stats = [
        'new_clients' => countNewClients($params['cronlastrun']),
        'new_orders' => countNewOrders($params['cronlastrun']),
        'revenue' => calculateDailyRevenue($params['cronlastrun']),
        'tickets_opened' => countTicketsOpened($params['cronlastrun']),
        'tickets_resolved' => countTicketsResolved($params['cronlastrun'])
    ];
    
    // 2. Store statistics
    insert_query('tbl_daily_stats', [
        'date' => date('Y-m-d'),
        'stats' => json_encode($stats),
        'created_at' => date('Y-m-d H:i:s')
    ]);
    
    // 3. Send daily summary to admin
    sendAdminNotification('email', [
        'subject' => 'Daily Summary: ' . date('Y-m-d'),
        'message' => "New Clients: {$stats['new_clients']}\n" .
                     "New Orders: {$stats['new_orders']}\n" .
                     "Revenue: $" . number_format($stats['revenue'], 2) . "\n" .
                     "Tickets: {$stats['tickets_opened']} opened, " .
                     "{$stats['tickets_resolved']} resolved"
    ]);
    
    // 4. Update trend data for charts
    updateTrendData($stats);
    
    return $params;
});
```

## Automated Actions

```php
<?php
add_hook('DailyCronJob', 1, function(array $params) {
    // 1. Check for expiring SSL certificates
    $expiringSSL = full_query("
        SELECT * FROM tblssl
        WHERE expiry_date BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL 30 DAY)
    ");
    
    foreach ($expiringSSL as $cert) {
        sendSSLRenewalReminder($cert['id']);
        if ($cert['auto_renew']) {
            initiateSSLAutoRenew($cert['id']);
        }
    }
    
    // 2. Process affiliate commissions
    processAffiliateCommissions();
    
    // 3. Check for unused services (potential churn)
    $unusedServices = findLongInactiveServices(30); // 30 days
    foreach ($unusedServices as $serviceId) {
        sendUsageAlert($serviceId);
    }
    
    // 4. Back up critical data
    triggerDailyBackup('database');
    
    // 5. Sync with external systems
    syncToExternalAnalytics();
    
    return $params;
});
```

## Use Cases

- **Automated Billing**: Process suspensions, reminders
- **Maintenance**: Database cleanup, caching
- **Reporting**: Generate daily statistics
- **Integrations**: Sync with external systems
- **Monitoring**: Track system health

## Notes

- Runs once daily during cron execution
- Long-running tasks should be batched
- Consider using `HourlyCronJob` for more frequent tasks
- Monitor execution time to avoid timeouts

## Related Hooks

- `HourlyCronJob` - For hourly tasks
- `InvoicePaid` - For payment processing
- `DailyCronJob` alternative naming in some versions

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Cron Configuration](../whmcs-cron-setup.md)