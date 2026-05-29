# WHMCS Automation Integration

Complete guide for automating WHMCS operations with external systems.

## Overview

Set up automated workflows connecting WHMCS with external services.

## Cron Job Integration

### External Cron Trigger

```php
<?php
/**
 * External cron endpoint
 */
function handleExternalCron(): void
{
    // Verify request
    $secret = $_GET['secret'] ?? '';
    if ($secret !== CRON_SECRET) {
        http_response_code(401);
        exit('Unauthorized');
    }
    
    header('Content-Type: text/plain');
    
    echo "WHMCS Automation Cron - " . date('Y-m-d H:i:s') . "\n";
    echo str_repeat('-', 50) . "\n\n";
    
    // Run different automation tasks
    $results = [];
    
    // Sync with CRM
    echo "Running CRM sync...\n";
    $results['crm_sync'] = performFullCRMSync();
    echo "  Synced: {$results['crm_sync']['synced']}, Failed: {$results['crm_sync']['failed']}\n";
    
    // Process email queue
    echo "Processing email queue...\n";
    $results['emails'] = processEmailQueue();
    echo "  Processed: {$results['emails']} emails\n";
    
    // Sync DNS
    echo "Syncing DNS records...\n";
    $results['dns'] = syncAllDomainsDNS();
    echo "  Synced: {$results['dns']['synced']} domains\n";
    
    // Check service health
    echo "Checking service health...\n";
    $results['health'] = checkServicesHealth();
    echo "  Healthy: {$results['health']['healthy']}, Issues: {$results['health']['issues']}\n";
    
    // Log results
    logActivity('External cron completed: ' . json_encode($results));
    
    echo "\n" . str_repeat('-', 50) . "\n";
    echo "Cron completed at " . date('Y-m-d H:i:s') . "\n";
}
```

### Scheduled Tasks

```php
<?php
/**
 * WHMCS cron job tasks
 */
class AutomationTasks
{
    /**
     * Daily service health check
     */
    public static function dailyHealthCheck(): array
    {
        $results = [
            'services_checked' => 0,
            'issues_found' => 0,
            'notifications_sent' => 0,
        ];
        
        $services = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->get();
        
        foreach ($services as $service) {
            $results['services_checked']++;
            
            // Simulate health check
            $health = self::checkServiceHealth($service);
            
            if (!$health['healthy']) {
                $results['issues_found']++;
                
                // Log issue
                Capsule::table('mod_service_health')->insert([
                    'service_id' => $service->id,
                    'check_status' => 'failed',
                    'error_message' => $health['error'],
                    'checked_at' => date('Y-m-d H:i:s'),
                ]);
                
                // Notify admin
                if ($health['critical']) {
                    self::notifyAdmin($service, $health['error']);
                    $results['notifications_sent']++;
                }
            }
        }
        
        return $results;
    }
    
    /**
     * Check individual service health
     */
    private static function checkServiceHealth($service): array
    {
        $api = new YourModuleAPI(getServerConfig($service->server));
        
        try {
            $status = $api->getServiceStatus($service->account_id);
            
            return [
                'healthy' => $status['status'] === 'active',
                'critical' => false,
                'details' => $status,
            ];
        } catch (Exception $e) {
            return [
                'healthy' => false,
                'critical' => strpos($e->getMessage(), 'timeout') !== false,
                'error' => $e->getMessage(),
            ];
        }
    }
    
    /**
     * Notify admin of issues
     */
    private static function notifyAdmin($service, string $error): void
    {
        // Send email to admin
        sendAdminNotification([
            'subject' => "Service Health Alert: {$service->domain}",
            'message' => "Service {$service->domain} is experiencing issues: {$error}",
        ]);
    }
    
    /**
     * Monthly usage reporting
     */
    public static function generateMonthlyUsageReport(): array
    {
        $startDate = date('Y-m-01', strtotime('-1 month'));
        $endDate = date('Y-m-t', strtotime('-1 month'));
        
        $services = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->get();
        
        $report = [
            'period' => ['start' => $startDate, 'end' => $endDate],
            'services' => [],
            'generated_at' => date('Y-m-d H:i:s'),
        ];
        
        foreach ($services as $service) {
            $usage = self::getServiceUsage($service, $startDate, $endDate);
            $report['services'][] = [
                'service_id' => $service->id,
                'domain' => $service->domain,
                'usage' => $usage,
            ];
        }
        
        // Store report
        Capsule::table('mod_usage_reports')->insert([
            'period_start' => $startDate,
            'period_end' => $endDate,
            'report_data' => json_encode($report),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        return $report;
    }
    
    /**
     * Get service usage for period
     */
    private static function getServiceUsage($service, string $start, string $end): array
    {
        return [
            'bandwidth_used' => 0, // Calculate from logs
            'storage_used' => 0,
            'api_calls' => 0,
        ];
    }
}
```

## Webhook Automation

```php
<?php
/**
 * Automated webhook triggers
 */
class WebhookAutomation
{
    /**
     * Trigger workflow on service change
     */
    public static function onServiceChange(array $serviceData, string $event): void
    {
        $workflows = self::getWorkflowsForEvent($event);
        
        foreach ($workflows as $workflow) {
            self::executeWorkflow($workflow, $serviceData);
        }
    }
    
    /**
     * Get active workflows for event
     */
    private static function getWorkflowsForEvent(string $event): array
    {
        return Capsule::table('mod_workflows')
            ->where('trigger_event', $event)
            ->where('enabled', 1)
            ->get()
            ->toArray();
    }
    
    /**
     * Execute workflow
     */
    private static function executeWorkflow(array $workflow, array $data): void
    {
        $actions = json_decode($workflow['actions'], true);
        
        foreach ($actions as $action) {
            switch ($action['type']) {
                case 'webhook':
                    self::triggerWebhook($action['url'], $data);
                    break;
                case 'email':
                    self::sendEmail($action['to'], $action['subject'], $action['body']);
                    break;
                case 'api':
                    self::callAPI($action['endpoint'], $data);
                    break;
            }
        }
    }
    
    /**
     * Trigger external webhook
     */
    private static function triggerWebhook(string $url, array $data): void
    {
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
        ]);
        curl_exec($ch);
        curl_close($ch);
    }
}
```

## Task Queue

```php
<?php
/**
 * Background task queue
 */
class TaskQueue
{
    /**
     * Add task to queue
     */
    public static function add(string $type, array $data, int $priority = 0): int
    {
        return Capsule::table('mod_task_queue')->insertGetId([
            'task_type' => $type,
            'task_data' => json_encode($data),
            'priority' => $priority,
            'status' => 'pending',
            'scheduled_at' => date('Y-m-d H:i:s'),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    /**
     * Process queue
     */
    public static function process(int $batchSize = 100): int
    {
        $tasks = Capsule::table('mod_task_queue')
            ->where('status', 'pending')
            ->where('scheduled_at', '<=', date('Y-m-d H:i:s'))
            ->orderBy('priority', 'desc')
            ->orderBy('created_at', 'asc')
            ->limit($batchSize)
            ->get();
        
        $processed = 0;
        
        foreach ($tasks as $task) {
            try {
                self::executeTask($task);
                Capsule::table('mod_task_queue')
                    ->where('id', $task->id)
                    ->update([
                        'status' => 'completed',
                        'completed_at' => date('Y-m-d H:i:s'),
                    ]);
                $processed++;
            } catch (Exception $e) {
                Capsule::table('mod_task_queue')
                    ->where('id', $task->id)
                    ->update([
                        'attempts' => $task->attempts + 1,
                        'last_error' => $e->getMessage(),
                        'status' => $task->attempts >= 3 ? 'failed' : 'pending',
                    ]);
            }
        }
        
        return $processed;
    }
    
    /**
     * Execute task
     */
    private static function executeTask($task): void
    {
        $data = json_decode($task->task_data, true);
        
        switch ($task->task_type) {
            case 'sync_crm':
                syncClientToCRM($data['client_id']);
                break;
            case 'send_notification':
                sendNotification($data['type'], $data);
                break;
            case 'cleanup':
                performCleanup($data['target']);
                break;
        }
    }
}
```

## Workflow Builder

```php
<?php
/**
 * Create automation workflow
 */
function createWorkflow(string $name, string $triggerEvent, array $actions): int
{
    return Capsule::table('mod_workflows')->insertGetId([
        'name' => $name,
        'trigger_event' => $triggerEvent,
        'actions' => json_encode($actions),
        'enabled' => 1,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Available workflow events
 */
const WORKFLOW_EVENTS = [
    'ServiceCreated' => 'Service provisioned',
    'ServiceSuspended' => 'Service suspended',
    'ServiceTerminated' => 'Service terminated',
    'InvoicePaid' => 'Invoice marked as paid',
    'InvoiceOverdue' => 'Invoice became overdue',
    'ClientRegistered' => 'New client registered',
    'TicketCreated' => 'Support ticket opened',
    'DomainRegistered' => 'Domain registered',
    'DomainExpired' => 'Domain expired',
];

/**
 * Example: New service automation
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    WebhookAutomation::onServiceChange($vars, 'ServiceCreated');
});
```

## Best Practices

1. **Idempotent tasks** - Handle repeated executions safely
2. **Error handling** - Log and retry failed tasks
3. **Rate limiting** - Respect external API limits
4. **Monitor queues** - Track task processing
5. **Timeouts** - Set appropriate limits
6. **Logging** - Record all automation actions

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-integration-webhooks.md](whmcs-integration-webhooks.md)
