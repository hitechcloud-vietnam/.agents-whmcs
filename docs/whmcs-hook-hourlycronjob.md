# WHMCS HourlyCronJob Hook Reference

## Overview

The `HourlyCronJob` hook fires during WHMCS hourly cron execution. This hook is ideal for time-sensitive tasks that need more frequent processing than the daily cron.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `cronlastrun` | string | Previous cron run timestamp |
| `crontnextrun` | string | Next cron run timestamp |
| `croncurrentrun` | string | Current cron run timestamp |

## Example Implementation

```php
<?php
add_hook('HourlyCronJob', 1, function(array $params) {
    // Log cron execution
    logActivity("Hourly cron executed at " . date('Y-m-d H:i:s'));
    
    // Run lightweight maintenance
    checkSystemHealth();
    
    return $params;
});
```

## Order Processing

```php
<?php
add_hook('HourlyCronJob', 1, function(array $params) {
    // 1. Process pending orders
    $pendingOrders = full_query("
        SELECT * FROM tblorders 
        WHERE status = 'Pending' 
        AND created_at < DATE_SUB(NOW(), INTERVAL 1 HOUR)
    ");
    
    foreach ($pendingOrders as $order) {
        // Verify payment and fulfill if applicable
        verifyAndFulfillOrder($order['id']);
    }
    
    // 2. Retry failed provisioning
    $failedProvisioning = full_query("
        SELECT * FROM tblhosting 
        WHERE status = 'Provisioning' 
        AND provisioning_retry < 3
        AND created_at < DATE_SUB(NOW(), INTERVAL 30 MINUTE)
    ");
    
    foreach ($failedProvisioning as $service) {
        retryServiceProvisioning($service['id']);
    }
    
    // 3. Process pending cancellations
    processScheduledCancellations();
    
    return $params;
});
```

## Payment Processing

```php
<?php
add_hook('HourlyCronJob', 1, function(array $params) {
    // 1. Check for pending credit card payments
    $pendingCC = full_query("
        SELECT * FROM tblinvoicepayments 
        WHERE gateway = 'creditcard' 
        AND status = 'pending'
        AND created_at < DATE_SUB(NOW(), INTERVAL 15 MINUTE)
    ");
    
    foreach ($pendingCC as $payment) {
        checkCCPaymentStatus($payment['id']);
    }
    
    // 2. Retry failed refunds
    $failedRefunds = full_query("
        SELECT * FROM tbl_refunds 
        WHERE status = 'failed' 
        AND retry_count < 3
    ");
    
    foreach ($failedRefunds as $refund) {
        retryRefund($refund['id']);
    }
    
    // 3. Process batch payments
    processPendingBatchPayments();
    
    return $params;
});
```

## Monitoring and Alerts

```php
<?php
add_hook('HourlyCronJob', 1, function(array $params) {
    // 1. Check server status
    $servers = full_query("SELECT * FROM tblservers WHERE disabled = 0");
    foreach ($servers as $server) {
        $status = checkServerUptime($server['id']);
        if (!$status['online']) {
            sendServerDownAlert($server);
        }
    }
    
    // 2. Monitor disk space on servers
    $lowDiskServers = getServersWithLowDisk();
    if (count($lowDiskServers) > 0) {
        sendAdminNotification('system', [
            'subject' => 'Low Disk Space Warning',
            'message' => 'Servers with low disk space: ' . implode(', ', $lowDiskServers)
        ]);
    }
    
    // 3. Check for orphaned processes
    cleanupOrphanedProcesses();
    
    // 4. Update real-time statistics
    updateRealTimeStats();
    
    return $params;
});
```

## Queue Processing

```php
<?php
add_hook('HourlyCronJob', 1, function(array $params) {
    // 1. Process email queue
    $emailQueue = getEmailQueue(50); // Process 50 per hour
    foreach ($emailQueue as $email) {
        sendQueuedEmail($email['id']);
    }
    
    // 2. Process webhook queue
    $webhookQueue = getWebhookQueue(100);
    foreach ($webhookQueue as $webhook) {
        deliverWebhook($webhook['id']);
    }
    
    // 3. Process provisioning queue
    processProvisioningQueue();
    
    // 4. Sync data to external systems
    syncQueuedData();
    
    return $params;
});
```

## Use Cases

- **Order Processing**: Handle pending orders more frequently
- **Payment Verification**: Check payment statuses
- **Monitoring**: Server and system health checks
- **Queue Processing**: Email, webhooks, provisioning
- **Caching**: Refresh cached data

## Notes

- Runs every hour during cron execution
- Keep tasks lightweight to avoid timeouts
- Use database transactions for data integrity
- Combine with `DailyCronJob` for comprehensive automation

## Related Hooks

- `DailyCronJob` - For daily tasks
- `OrderPaid` - For order payment processing
- `InvoicePaid` - For payment processing

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Cron Configuration](../whmcs-cron-setup.md)