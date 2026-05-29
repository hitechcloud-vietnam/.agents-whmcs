# WHMCS Batch Operations Workflow

## Overview
This workflow implements batch processing capabilities for WHMCS to handle bulk operations efficiently.

## Prerequisites
- WHMCS installation with custom hooks
- PHP 7.4+ for modern syntax
- Adequate server resources for batch processing

## Step-by-Step Process

### Step 1: Understand Batch Processing Architecture
```
BATCH PROCESSING FLOW:
1. Collect items requiring processing
2. Group items by type/priority
3. Process in controlled batches
4. Track progress and results
5. Handle failures and retries
6. Generate completion report
```

### Step 2: Create Batch Processing Manager
```php
<?php
// /includes/hooks/batch_processing.php

/**
 * Batch Processing System for WHMCS
 */

class BatchProcessor
{
    private $batchSize;
    private $delayBetweenBatches;
    private $results = [];
    private $errors = [];

    public function __construct($batchSize = 50, $delayBetweenBatches = 1)
    {
        $this->batchSize = $batchSize;
        $this->delayBetweenBatches = $delayBetweenBatches;
    }

    public function process($items, callable $processor, $options = [])
    {
        $options = array_merge([
            'stopOnError' => false,
            'logProgress' => true,
            'maxErrors' => 100
        ], $options);

        $total = count($items);
        $processed = 0;
        $batches = array_chunk($items, $this->batchSize);
        $batchNum = 0;

        foreach ($batches as $batch) {
            $batchNum++;
            $batchStart = microtime(true);

            foreach ($batch as $item) {
                try {
                    $result = $processor($item);

                    $this->results[] = [
                        'item' => $item,
                        'result' => $result,
                        'status' => 'success',
                        'timestamp' => date('Y-m-d H:i:s')
                    ];

                    $processed++;

                    if ($options['logProgress']) {
                        logBatchProgress($this->batchSize, $batchNum, $processed, $total);
                    }
                } catch (Exception $e) {
                    $this->handleError($item, $e, $options);
                    $processed++;

                    if ($options['stopOnError']) {
                        break 2;
                    }
                }

                if (count($this->errors) >= $options['maxErrors']) {
                    logActivity("Batch processing stopped: max errors reached");
                    break 2;
                }
            }

            // Delay between batches
            if ($batchNum < count($batches)) {
                sleep($this->delayBetweenBatches);
            }
        }

        return $this->generateReport();
    }

    private function handleError($item, Exception $e, $options)
    {
        $this->errors[] = [
            'item' => $item,
            'error' => $e->getMessage(),
            'timestamp' => date('Y-m-d H:i:s')
        ];

        logBatchError($item, $e);
    }

    private function generateReport()
    {
        return [
            'total' => count($this->results) + count($this->errors),
            'successful' => count($this->results),
            'failed' => count($this->errors),
            'errors' => $this->errors,
            'results' => $this->results
        ];
    }
}

function logBatchProgress($batchSize, $batchNum, $processed, $total)
{
    $percent = round(($processed / $total) * 100, 1);
    logActivity("Batch {$batchNum}: {$processed}/{$total} ({$percent}%)");
}

function logBatchError($item, Exception $e)
{
    logActivity("Batch error: " . json_encode($item) . " - " . $e->getMessage());
}
```

### Step 3: Batch Client Operations
```php
/**
 * Batch client status updates
 */
add_hook('DailyCronJob', 1, function($vars) {
    // Get clients needing status review
    $clients = Capsule::table('tblclients')
        ->where('status', 'Active')
        ->where('lastlogin', '<', date('Y-m-d', strtotime('-365 days')))
        ->get();

    $processor = new BatchProcessor(25, 2);

    return $processor->process($clients->toArray(), function($client) {
        // Check client activity
        $recentInvoices = Capsule::table('tblinvoices')
            ->where('userid', $client->id)
            ->where('date', '>', date('Y-m-d', strtotime('-180 days')))
            ->count();

        $recentTickets = Capsule::table('tbltickets')
            ->where('userid', $client->id)
            ->where('date', '>', date('Y-m-d', strtotime('-180 days')))
            ->count();

        // Inactive client
        if ($recentInvoices == 0 && $recentTickets == 0) {
            Capsule::table('tblclients')
                ->where('id', $client->id)
                ->update(['status' => 'Inactive']);

            return ['action' => 'marked_inactive', 'reason' => 'No activity'];
        }

        return ['action' => 'reviewed', 'reason' => 'Active'];
    });
});
```

### Step 4: Batch Service Operations
```php
/**
 * Batch service provisioning checks
 */
add_hook('DailyCronJob', 1, function($vars) {
    // Get all pending services
    $pendingServices = Capsule::table('tblhosting')
        ->where('domainstatus', 'Pending')
        ->where('servertype', '!=', '')
        ->get();

    $processor = new BatchProcessor(10, 3);

    return $processor->process($pendingServices->toArray(), function($service) {
        $server = Capsule::table('tblservers')
            ->where('id', $service->server)
            ->first();

        if (!$server) {
            return ['action' => 'skipped', 'reason' => 'No server assigned'];
        }

        // Check if already provisioned
        $result = provisionService($service, $server);

        if ($result['success']) {
            Capsule::table('tblhosting')
                ->where('id', $service->id)
                ->update(['domainstatus' => 'Active']);

            sendServiceActiveEmail($service->userid, $service->id);

            return ['action' => 'activated', 'server' => $server->name];
        }

        return ['action' => 'failed', 'error' => $result['error']];
    });
});

/**
 * Batch service renewal processing
 */
add_hook('DailyCronJob', 1, function($vars) {
    $today = date('Y-m-d');
    $nextWeek = date('Y-m-d', strtotime('+7 days'));

    // Get services due for renewal
    $renewals = Capsule::table('tblhosting')
        ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
        ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
        ->where('tblhosting.nextduedate', '<=', $nextWeek)
        ->where('tblhosting.domainstatus', 'Active')
        ->whereIn('tblhosting.billingcycle', ['Monthly', 'Quarterly', 'Semi-Annual', 'Annual'])
        ->get();

    $processor = new BatchProcessor(30, 1);

    return $processor->process($renewals->toArray(), function($service) use ($today) {
        // Check if renewal invoice exists
        $existingInvoice = Capsule::table('tblinvoices')
            ->where('userid', $service->userid)
            ->where('status', '!=', 'Paid')
            ->whereRaw("CAST(items AS CHAR) LIKE '%\"relid\":{$service->id}%'")
            ->first();

        if (!$existingInvoice) {
            // Create renewal invoice
            $invoiceId = createRenewalInvoice($service);

            return ['action' => 'invoice_created', 'invoice_id' => $invoiceId];
        }

        return ['action' => 'skipped', 'reason' => 'Invoice exists'];
    });
});
```

### Step 5: Batch Invoice Operations
```php
/**
 * Batch invoice generation
 */
add_hook('DailyCronJob', 1, function($vars) {
    $today = date('Y-m-d');

    // Get all services due for billing
    $dueServices = Capsule::table('tblhosting')
        ->where('nextduedate', '<=', $today)
        ->where('domainstatus', 'Active')
        ->whereIn('billingcycle', ['Monthly', 'Quarterly', 'Semi-Annual', 'Annual'])
        ->whereNotExists(function($q) {
            $q->select(Capsule::raw(1))
                ->from('tblinvoices')
                ->whereRaw("CAST(items AS CHAR) LIKE CONCAT('%\"relid\":', tblhosting.id, '%')")
                ->where('status', '!=', 'Paid');
        })
        ->get();

    $processor = new BatchProcessor(20, 2);

    return $processor->process($dueServices->toArray(), function($service) {
        // Create invoice
        $invoiceData = [
            'userid' => $service->userid,
            'sendinvoice' => true,
            'autoapplycredit' => true
        ];

        $result = localApi('CreateInvoice', $invoiceData);

        if ($result['invoiceid']) {
            // Add service item
            localApi('AddInvoiceItem', [
                'invoiceid' => $result['invoiceid'],
                'type' => 'Hosting',
                'relid' => $service->id,
                'description' => getServiceDescription($service),
                'amount' => getServicePrice($service)
            ]);

            return ['action' => 'created', 'invoice_id' => $result['invoiceid']];
        }

        return ['action' => 'failed', 'reason' => $result['error'] ?? 'Unknown'];
    });
});

/**
 * Batch invoice reminders
 */
add_hook('DailyCronJob', 1, function($vars) {
    $overdueInvoices = Capsule::table('tblinvoices')
        ->where('status', 'Overdue')
        ->where('duedate', '<', date('Y-m-d', strtotime('-3 days')))
        ->where('reminder_sent', '!=', date('Y-m-d'))
        ->get();

    $processor = new BatchProcessor(50, 1);

    return $processor->process($overdueInvoices->toArray(), function($invoice) {
        // Send reminder
        sendEmail($invoice->userid, 'Invoice Overdue Reminder', [
            'invoice_id' => $invoice->id,
            'amount' => $invoice->total,
            'due_date' => $invoice->duedate,
            'days_overdue' => daysOverdue($invoice->duedate)
        ]);

        // Mark as sent
        Capsule::table('tblinvoices')
            ->where('id', $invoice->id)
            ->update(['reminder_sent' => date('Y-m-d')]);

        return ['action' => 'reminder_sent', 'client_id' => $invoice->userid];
    });
});
```

### Step 6: Batch Domain Operations
```php
/**
 * Batch domain status sync
 */
add_hook('HourlyCronJob', 1, function($vars) {
    // Get domains pending sync
    $domains = Capsule::table('tbldomains')
        ->where('status', 'Active')
        ->where('registrationdate', '!=', '0000-00-00')
        ->whereRaw('UNIX_TIMESTAMP(last_updated) < UNIX_TIMESTAMP(NOW()) - 86400')
        ->limit(100)
        ->get();

    $processor = new BatchProcessor(25, 2);

    return $processor->process($domains->toArray(), function($domain) {
        $registrar = Capsule::table('tblregistrars')
            ->where('registrar', $domain->registrar)
            ->first();

        if (!$registrar || !$registrar->setting('AutoSync')) {
            return ['action' => 'skipped', 'reason' => 'Sync disabled'];
        }

        // Sync domain status
        $syncResult = syncDomainWithRegistrar($domain, $registrar);

        if ($syncResult['success']) {
            Capsule::table('tbldomains')
                ->where('id', $domain->id)
                ->update([
                    'expirydate' => $syncResult['expiry'],
                    'status' => $syncResult['status'],
                    'last_updated' => date('Y-m-d H:i:s')
                ]);

            return ['action' => 'synced', 'expiry' => $syncResult['expiry']];
        }

        return ['action' => 'failed', 'error' => $syncResult['error']];
    });
});

/**
 * Batch domain renewal check
 */
add_hook('DailyCronJob', 1, function($vars) {
    $warningDate = date('Y-m-d', strtotime('+30 days'));
    $urgentDate = date('Y-m-d', strtotime('+7 days'));

    // Get domains expiring soon
    $expiringDomains = Capsule::table('tbldomains')
        ->where('status', 'Active')
        ->where('expirydate', '<=', $warningDate)
        ->where('expirydate', '>=', date('Y-m-d'))
        ->get();

    $processor = new BatchProcessor(30, 1);

    return $processor->process($expiringDomains->toArray(), function($domain) use ($urgentDate) {
        $daysUntilExpiry = daysUntil($domain->expirydate);

        // Check if already notified
        $notificationKey = "renewal_notice_{$domain->id}_{$daysUntilExpiry}";

        if (alreadyNotified($notificationKey)) {
            return ['action' => 'skipped', 'reason' => 'Already notified'];
        }

        // Determine email template
        if ($domain->expirydate <= $urgentDate) {
            $template = 'Domain Expiry Urgent';
            $priority = 'high';
        } else {
            $template = 'Domain Expiry Notice';
            $priority = 'normal';
        }

        sendEmail($domain->userid, $template, [
            'domain' => $domain->domain,
            'expiry_date' => $domain->expirydate,
            'days_remaining' => $daysUntilExpiry,
            'renewal_url' => getRenewalUrl($domain->id)
        ]);

        markNotified($notificationKey);

        return ['action' => 'notified', 'days' => $daysUntilExpiry, 'priority' => $priority];
    });
});
```

### Step 7: Batch Import/Export Operations
```php
/**
 * Batch CSV import
 */
function batchImportFromCSV($filePath, $options = [])
{
    $options = array_merge([
        'delimiter' => ',',
        'enclosure' => '"',
        'hasHeader' => true,
        'batchSize' => 100
    ], $options);

    $handle = fopen($filePath, 'r');

    if ($options['hasHeader']) {
        fgetcsv($handle, 0, $options['delimiter'], $options['enclosure']);
    }

    $processor = new BatchProcessor($options['batchSize'], 0);
    $rows = [];

    while (($row = fgetcsv($handle, 0, $options['delimiter'], $options['enclosure'])) !== false) {
        $rows[] = $row;

        if (count($rows) >= $options['batchSize']) {
            $processor->process($rows, $options['processor'] ?? 'processImportRow');
            $rows = [];
        }
    }

    // Process remaining rows
    if (!empty($rows)) {
        $processor->process($rows, $options['processor'] ?? 'processImportRow');
    }

    fclose($handle);

    return $processor->generateReport();
}

/**
 * Batch CSV export
 */
function batchExportToCSV($query, $filePath, $options = [])
{
    $options = array_merge([
        'delimiter' => ',',
        'enclosure' => '"',
        'batchSize' => 1000,
        'includeHeader' => true,
        'columns' => null
    ], $options);

    $handle = fopen($filePath, 'w');

    // Get first batch for header
    $firstBatch = Capsule::table($query['table'])
        ->select($options['columns'] ?? '*')
        ->limit($options['batchSize'])
        ->offset(0)
        ->get();

    if ($options['includeHeader'] && !empty($firstBatch)) {
        fputcsv($handle, array_keys((array)$firstBatch[0]), $options['delimiter'], $options['enclosure']);
    }

    $offset = 0;

    while (true) {
        $batch = Capsule::table($query['table'])
            ->select($options['columns'] ?? '*')
            ->limit($options['batchSize'])
            ->offset($offset)
            ->get();

        if ($batch->isEmpty()) {
            break;
        }

        foreach ($batch as $row) {
            fputcsv($handle, (array)$row, $options['delimiter'], $options['enclosure']);
        }

        $offset += $options['batchSize'];
    }

    fclose($handle);

    return ['file' => $filePath, 'size' => filesize($filePath)];
}
```

### Step 8: Batch Report Generation
```php
/**
 * Generate batch reports
 */
add_hook('MonthlyCronJob', 1, function($vars) {
    $reportTypes = [
        'monthly_revenue',
        'new_clients',
        'churned_services',
        'top_products',
        'payment_methods'
    ];

    $results = [];

    foreach ($reportTypes as $reportType) {
        $generator = "generate" . str_replace('_', '', ucwords($reportType, '_')) . "Report";

        $results[$reportType] = $generator();
    }

    // Store reports
    foreach ($results as $type => $data) {
        Capsule::table('mod_reports')->insert([
            'report_type' => $type,
            'period' => date('Y-m'),
            'data' => json_encode($data),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    // Send summary to admin
    sendAdminEmail('Monthly Reports Generated', [
        'reports' => array_keys($results),
        'date' => date('Y-m-d')
    ]);

    return $results;
});

function generateMonthlyRevenueReport()
{
    $startDate = date('Y-m-01');
    $endDate = date('Y-m-t');

    return Capsule::select("
        SELECT
            DATE(datepaid) as date,
            SUM(total) as revenue,
            COUNT(*) as transactions,
            paymentmethod
        FROM tblinvoices
        WHERE datepaid BETWEEN ? AND ?
        AND status = 'Paid'
        GROUP BY DATE(datepaid), paymentmethod
        ORDER BY date
    ", [$startDate, $endDate]);
}
```

## Batch Processing Best Practices

1. **Start Small** - Test with smaller batches first
2. **Monitor Resources** - Watch CPU/memory usage
3. **Implement Logging** - Track progress and errors
4. **Use Transactions** - Ensure data consistency
5. **Set Timeouts** - Prevent runaway processes
6. **Schedule Wisely** - Run during off-peak hours

## Batch Size Recommendations

| Operation Type | Recommended Batch Size | Delay |
|----------------|----------------------|-------|
| Database updates | 50-100 | 1-2s |
| API calls | 10-25 | 2-5s |
| Email sends | 100-200 | 0.5-1s |
| File operations | 25-50 | 1-2s |
| Server commands | 5-10 | 3-5s |

## Related Workflows
- [WHMCS Loop Automation](./whmcs-loop-automation.md)
- [WHMCS Queue Processing](./whmcs-queue-processing.md)
- [WHMCS Scheduled Tasks](./whmcs-scheduled-tasks.md)