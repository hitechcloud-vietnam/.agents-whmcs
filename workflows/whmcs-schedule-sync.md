# WHMCS Scheduled Synchronization Workflow

## Purpose
Set up and manage scheduled data synchronization tasks for WHMCS.

## Prerequisites
- WHMCS installation
- Cron access
- Sync scripts configured

## Step-by-Step Process

### Step 1: Create Scheduled Sync Manager

**Create hooks/scheduled_sync.php:**
```php
<?php
/**
 * WHMCS Scheduled Synchronization Manager
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class ScheduledSync {
    
    private $syncConfigs = [];
    private $syncLog = [];
    
    public function __construct() {
        $this->loadConfigs();
    }
    
    /**
     * Load sync configurations
     */
    private function loadConfigs() {
        $this->syncConfigs = [
            'daily_clients' => [
                'enabled' => true,
                'schedule' => '0 2 * * *', // 2 AM daily
                'type' => 'export',
                'destination' => 'ftp',
                'entity' => 'clients',
                'filters' => ['include_inactive' => true]
            ],
            'hourly_services' => [
                'enabled' => true,
                'schedule' => '0 * * * *', // Every hour
                'type' => 'bidirectional',
                'destination' => 'api',
                'entity' => 'services'
            ],
            'daily_backup' => [
                'enabled' => true,
                'schedule' => '30 3 * * *', // 3:30 AM daily
                'type' => 'backup',
                'destination' => 'cloud',
                'entity' => 'all'
            ],
            'weekly_reports' => [
                'enabled' => true,
                'schedule' => '0 6 * * 1', // Monday 6 AM
                'type' => 'report',
                'destination' => 'email',
                'entity' => 'summary'
            ]
        ];
    }
    
    /**
     * Check if task should run
     */
    public function shouldRun($taskId) {
        if (!isset($this->syncConfigs[$taskId])) {
            return false;
        }
        
        $config = $this->syncConfigs[$taskId];
        
        if (!$config['enabled']) {
            return false;
        }
        
        // Check last run time
        $lastRun = $this->getLastRunTime($taskId);
        
        if (!$lastRun) {
            return true;
        }
        
        // Parse schedule and check
        return $this->isScheduleDue($config['schedule'], $lastRun);
    }
    
    /**
     * Check if schedule is due
     */
    private function isScheduleDue($cron, $lastRun) {
        $parts = explode(' ', $cron);
        
        if (count($parts) !== 5) {
            return false;
        }
        
        [$minute, $hour, $dayOfMonth, $month, $dayOfWeek] = $parts;
        
        $now = Carbon::now();
        $last = Carbon::parse($lastRun);
        
        // Check minute
        if ($minute !== '*' && (int)$minute !== (int)$now->format('i')) {
            return false;
        }
        
        // Check hour
        if ($hour !== '*' && (int)$hour !== (int)$now->format('H')) {
            return false;
        }
        
        // Check day of month
        if ($dayOfMonth !== '*' && (int)$dayOfMonth !== (int)$now->format('j')) {
            return false;
        }
        
        // Check month
        if ($month !== '*' && (int)$month !== (int)$now->format('n')) {
            return false;
        }
        
        // Check day of week
        if ($dayOfWeek !== '*' && (int)$dayOfWeek !== (int)$now->format('N')) {
            return false;
        }
        
        // Ensure not same hour as last run
        if ($last && $last->format('Y-m-d H') === $now->format('Y-m-d H')) {
            return false;
        }
        
        return true;
    }
    
    /**
     * Execute scheduled sync
     */
    public function executeSync($taskId) {
        if (!$this->shouldRun($taskId)) {
            return ['status' => 'skipped', 'reason' => 'Not due'];
        }
        
        $startTime = microtime(true);
        $result = [
            'task_id' => $taskId,
            'start_time' => Carbon::now()->toDateTimeString(),
            'status' => 'success',
            'details' => []
        ];
        
        try {
            $config = $this->syncConfigs[$taskId];
            
            switch ($config['type']) {
                case 'export':
                    $result['details'] = $this->executeExport($config);
                    break;
                case 'bidirectional':
                    $result['details'] = $this->executeBidirectional($config);
                    break;
                case 'backup':
                    $result['details'] = $this->executeBackup($config);
                    break;
                case 'report':
                    $result['details'] = $this->executeReport($config);
                    break;
            }
            
        } catch (Exception $e) {
            $result['status'] = 'error';
            $result['error'] = $e->getMessage();
        }
        
        $result['end_time'] = Carbon::now()->toDateTimeString();
        $result['duration'] = round(microtime(true) - $startTime, 2);
        
        // Update last run time
        $this->updateLastRunTime($taskId, $result['end_time']);
        
        // Log result
        $this->logSyncResult($result);
        
        return $result;
    }
    
    /**
     * Execute export sync
     */
    private function executeExport($config) {
        $exporter = new DataExport();
        
        switch ($config['entity']) {
            case 'clients':
                $result = $exporter->exportClients($config['filters'] ?? []);
                break;
            case 'services':
                $result = $exporter->exportServices();
                break;
            case 'invoices':
                $result = $exporter->exportInvoices();
                break;
            default:
                $result = $exporter->exportFullBackup();
        }
        
        // Upload to destination
        $this->uploadToDestination($result['path'], $config['destination']);
        
        return $result;
    }
    
    /**
     * Execute bidirectional sync
     */
    private function executeBidirectional($config) {
        $sync = new APISyncClient($this->getApiConfig());
        return $sync->fullSync();
    }
    
    /**
     * Execute backup sync
     */
    private function executeBackup($config) {
        $exporter = new DataExport();
        $result = $exporter->exportFullBackup();
        
        // Upload to cloud
        $this->uploadToCloud($result['path'], 'backup-' . date('Y-m-d'));
        
        // Clean old backups (keep last 30 days)
        $this->cleanOldBackups(30);
        
        return $result;
    }
    
    /**
     * Execute report generation
     */
    private function executeReport($config) {
        $report = $this->generateSummaryReport();
        
        // Send via email
        $this->sendReportEmail($report);
        
        return ['report_sent' => true];
    }
    
    /**
     * Upload to destination
     */
    private function uploadToDestination($file, $destination) {
        switch ($destination) {
            case 'ftp':
                $ftp = new FTPImport($this->getFtpConfig());
                $ftp->uploadFile($file);
                break;
            case 'sftp':
                $sftp = new SFTPSync($this->getSftpConfig());
                $sftp->uploadFile($file);
                break;
            case 'cloud':
                $cloud = new CloudStorageImport($this->getCloudConfig());
                $cloud->uploadFile($file, 'exports/' . basename($file));
                break;
        }
    }
    
    /**
     * Upload to cloud
     */
    private function uploadToCloud($file, $prefix) {
        $cloud = new CloudStorageImport($this->getCloudConfig());
        $cloud->uploadFile($file, $prefix . '/' . basename($file));
    }
    
    /**
     * Clean old backups
     */
    private function cleanOldBackups($days) {
        $cloud = new CloudStorageImport($this->getCloudConfig());
        $objects = $cloud->listObjects('backup-');
        
        $cutoff = Carbon::now()->subDays($days)->toDateString();
        
        foreach ($objects as $object) {
            $date = substr($object['key'], 8, 10); // Extract date from filename
            if ($date < $cutoff) {
                $cloud->deleteObject($object['key']);
            }
        }
    }
    
    /**
     * Generate summary report
     */
    private function generateSummaryReport() {
        $report = [
            'generated_at' => Carbon::now()->toDateTimeString(),
            'period' => Carbon::now()->subWeek()->toDateString() . ' to ' . Carbon::now()->toDateString(),
            'clients' => [
                'total' => Capsule::table('tblclients')->count(),
                'new_this_week' => Capsule::table('tblclients')
                    ->where('datecreated', '>=', Carbon::now()->subWeek()->toDateString())
                    ->count()
            ],
            'services' => [
                'total' => Capsule::table('tblhosting')->count(),
                'active' => Capsule::table('tblhosting')
                    ->where('domainstatus', 'Active')
                    ->count()
            ],
            'invoices' => [
                'total' => Capsule::table('tblinvoices')
                    ->where('date', '>=', Carbon::now()->subWeek()->toDateString())
                    ->count(),
                'paid' => Capsule::table('tblinvoices')
                    ->where('status', 'Paid')
                    ->where('date', '>=', Carbon::now()->subWeek()->toDateString())
                    ->count(),
                'total_amount' => Capsule::table('tblinvoices')
                    ->where('status', 'Paid')
                    ->where('date', '>=', Carbon::now()->subWeek()->toDateString())
                    ->sum('total')
            ],
            'sync_history' => $this->getRecentSyncHistory(10)
        ];
        
        return $report;
    }
    
    /**
     * Send report email
     */
    private function sendReportEmail($report) {
        $adminEmail = \WHMCS\Config\Setting::getValue('Email');
        $adminName = \WHMCS\Config\Setting::getValue('CompanyName');
        
        $subject = 'Weekly Sync Report - ' . Carbon::now()->format('Y-m-d');
        $body = $this->formatReportBody($report);
        
        sendEmail($subject, $adminEmail, ['body' => $body]);
    }
    
    /**
     * Format report body
     */
    private function formatReportBody($report) {
        $html = '<h2>Weekly Sync Report</h2>';
        $html .= '<p>Generated: ' . $report['generated_at'] . '</p>';
        $html .= '<h3>Summary</h3>';
        $html .= '<ul>';
        $html .= '<li>Total Clients: ' . $report['clients']['total'] . '</li>';
        $html .= '<li>New Clients (7 days): ' . $report['clients']['new_this_week'] . '</li>';
        $html .= '<li>Active Services: ' . $report['services']['active'] . '</li>';
        $html .= '<li>Paid Invoices (7 days): ' . $report['invoices']['paid'] . '</li>';
        $html .= '<li>Total Revenue (7 days): $' . number_format($report['invoices']['total_amount'], 2) . '</li>';
        $html .= '</ul>';
        
        return $html;
    }
    
    /**
     * Get last run time
     */
    private function getLastRunTime($taskId) {
        $setting = Capsule::table('tblsettings')
            ->where('setting', 'sync_last_run_' . $taskId)
            ->first();
        
        return $setting ? $setting->value : null;
    }
    
    /**
     * Update last run time
     */
    private function updateLastRunTime($taskId, $time) {
        Capsule::table('tblsettings')
            ->updateOrInsert(
                ['setting' => 'sync_last_run_' . $taskId],
                ['value' => $time]
            );
    }
    
    /**
     * Log sync result
     */
    private function logSyncResult($result) {
        $this->syncLog[] = $result;
    }
    
    /**
     * Get recent sync history
     */
    private function getRecentSyncHistory($limit = 10) {
        $logs = Capsule::table('mod_sync_logs')
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get();
        
        return $logs->toArray();
    }
    
    /**
     * Run all due syncs
     */
    public function runDueSyncs() {
        $results = [];
        
        foreach ($this->syncConfigs as $taskId => $config) {
            if ($this->shouldRun($taskId)) {
                $results[$taskId] = $this->executeSync($taskId);
            }
        }
        
        return $results;
    }
}
```

### Step 2: Set Up Cron Jobs

```php
<?php
/**
 * Main sync cron job
 */
add_hook('DailyCronJob', 1, function($vars) {
    $scheduler = new ScheduledSync();
    $results = $scheduler->runDueSyncs();
    
    $successCount = count(array_filter($results, function($r) {
        return $r['status'] === 'success';
    }));
    
    logActivity("Scheduled Sync: $successCount of " . count($results) . " tasks completed");
    
    return $results;
});
```

### Step 3: Monitor Sync Status

```php
<?php
/**
 * Get sync status dashboard data
 */
function getSyncDashboardData() {
    $scheduler = new ScheduledSync();
    $status = [];
    
    $tasks = ['daily_clients', 'hourly_services', 'daily_backup', 'weekly_reports'];
    
    foreach ($tasks as $taskId) {
        $lastRun = $scheduler->getLastRunTime($taskId);
        $isDue = $scheduler->shouldRun($taskId);
        
        $status[$taskId] = [
            'last_run' => $lastRun,
            'is_due' => $isDue,
            'last_status' => $scheduler->getLastStatus($taskId)
        ];
    }
    
    return $status;
}
```

## Best Practices
- Schedule syncs during off-peak hours
- Implement retry logic for failures
- Monitor sync performance
- Log all sync operations
- Test schedules thoroughly
- Use appropriate intervals
- Handle timezone correctly
- Set up alerts for failures
- Review sync history regularly
- Document sync schedules
