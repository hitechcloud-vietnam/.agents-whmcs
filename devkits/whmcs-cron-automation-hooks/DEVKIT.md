# WHMCS Cron Automation Hooks Module

## Overview
Cron job automation module with DailyCronJob and HourlyCronJob hooks for scheduled task execution.

## Module File: hooks.php

```php
<?php
/**
 * WHMCS Cron Automation Hooks Module
 * 
 * @package    WHMCS
 * @subpackage Modules
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: DailyCronJob
 * Triggered by WHMCS daily cron execution
 */
function whmcs_cron_automation_daily(array $params): array
{
    try {
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('CronAutomation', 'DailyCronJob', [], 'Daily cron started', '');

        // Record cron execution
        Capsule::table('mod_cron_executions')->insert([
            'cron_type' => 'daily',
            'started_at' => $now,
            'status' => 'running',
        ]);

        // Run daily tasks
        runSuspensionChecks($config);
        runRenewalNotifications($config);
        runServiceExpirationChecks($config);
        runUsageBillingCycle($config);
        runAffiliateCommissionProcessing($config);
        runUsageReportGeneration($config);
        runCleanupTasks($config);
        runStatisticsUpdate($config);

        // Update execution record
        Capsule::table('mod_cron_executions')
            ->where('cron_type', 'daily')
            ->where('started_at', $now)
            ->update([
                'completed_at' => date('Y-m-d H:i:s'),
                'status' => 'completed',
            ]);

        logModuleCall('CronAutomation', 'DailyCronJob', [], 'Daily cron completed', '');

    } catch (\Exception $e) {
        logModuleCall('CronAutomation', 'DailyCronJob Error', [], $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: HourlyCronJob
 * Triggered by WHMCS hourly cron execution
 */
function whmcs_cron_automation_hourly(array $params): array
{
    try {
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('CronAutomation', 'HourlyCronJob', [], 'Hourly cron started', '');

        // Record cron execution
        Capsule::table('mod_cron_executions')->insert([
            'cron_type' => 'hourly',
            'started_at' => $now,
            'status' => 'running',
        ]);

        // Run hourly tasks
        runTicketSLAChecks($config);
        runMonitoringChecks($config);
        runPendingWebhookRetries($config);
        runCacheCleanup($config);

        // Update execution record
        Capsule::table('mod_cron_executions')
            ->where('cron_type', 'hourly')
            ->where('started_at', $now)
            ->update([
                'completed_at' => date('Y-m-d H:i:s'),
                'status' => 'completed',
            ]);

        logModuleCall('CronAutomation', 'HourlyCronJob', [], 'Hourly cron completed', '');

    } catch (\Exception $e) {
        logModuleCall('CronAutomation', 'HourlyCronJob Error', [], $e->getMessage(), '');
    }

    return [];
}

/**
 * Run suspension checks for overdue invoices
 */
function runSuspensionChecks(array $config): void
{
    if (empty($config['auto_suspend'])) {
        return;
    }

    $suspendDays = $config['suspend_after_days'] ?? 7;
    
    $overdueInvoices = Capsule::table('tblinvoices')
        ->where('status', 'Overdue')
        ->where('duedate', '<', date('Y-m-d', strtotime('-' . $suspendDays . ' days')))
        ->get(['userid', 'id']);

    foreach ($overdueInvoices as $invoice) {
        $services = Capsule::table('tblhosting')
            ->where('userid', $invoice->userid)
            ->where('domainstatus', 'Active')
            ->get(['id', 'domain']);

        foreach ($services as $service) {
            $command = 'ModuleSuspend';
            $postData = [
                'serviceid' => $service->id,
                'suspendreason' => 'Overdue Invoice #' . $invoice->id,
            ];
            localAPI($command, $postData);
        }
    }
}

/**
 * Run renewal notifications
 */
function runRenewalNotifications(array $config): void
{
    $notificationDays = $config['renewal_notification_days'] ?? [30, 14, 7, 3, 1];

    foreach ($notificationDays as $days) {
        $targetDate = date('Y-m-d', strtotime('+' . $days . ' days'));
        
        $services = Capsule::table('tblhosting')
            ->where('nextduedate', $targetDate)
            ->where('domainstatus', 'Active')
            ->get(['id', 'userid', 'domain']);

        foreach ($services as $service) {
            $alreadyNotified = Capsule::table('mod_renewal_notifications')
                ->where('service_id', $service->id)
                ->where('days_notice', $days)
                ->exists();

            if (!$alreadyNotified) {
                $command = 'SendEmail';
                $postData = [
                    'id' => $service->id,
                    'type' => 'product',
                    'templatename' => 'Service Renewal Reminder',
                ];
                localAPI($command, $postData);

                Capsule::table('mod_renewal_notifications')->insert([
                    'service_id' => $service->id,
                    'days_notice' => $days,
                    'notified_at' => date('Y-m-d H:i:s'),
                ]);
            }
        }
    }
}

/**
 * Run service expiration checks
 */
function runServiceExpirationChecks(array $config): void
{
    $expireDays = $config['expire_warning_days'] ?? 7;
    $targetDate = date('Y-m-d', strtotime('+' . $expireDays . ' days'));

    $expiringServices = Capsule::table('tblhosting')
        ->where('nextduedate', $targetDate)
        ->where('domainstatus', 'Active')
        ->get();

    foreach ($expiringServices as $service) {
        Capsule::table('mod_service_expiration_log')->insert([
            'service_id' => $service->id,
            'expiry_date' => $service->nextduedate,
            'days_until_expiry' => $expireDays,
            'checked_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * Run usage billing cycle
 */
function runUsageBillingCycle(array $config): void
{
    if (empty($config['usage_billing'])) {
        return;
    }

    // Process usage-based services for billing
    $usageServices = Capsule::table('tblhosting')
        ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
        ->where('tblproducts.type', 'hosting')
        ->where('tblproducts.configoption1', 'paytype', 'configoptions')
        ->where('tblhosting.domainstatus', 'Active')
        ->get();

    foreach ($usageServices as $service) {
        $usage = calculateServiceUsage($service->id);
        
        Capsule::table('mod_usage_records')->insert([
            'service_id' => $service->id,
            'billing_date' => date('Y-m-d'),
            'usage_data' => json_encode($usage),
            'calculated_amount' => $usage['estimated_cost'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * Calculate service usage
 */
function calculateServiceUsage(int $serviceId): array
{
    $usage = [
        'bandwidth_gb' => 0,
        'storage_gb' => 0,
        'api_calls' => 0,
        'estimated_cost' => 0,
    ];

    // Get from monitoring data
    $lastRecord = Capsule::table('mod_usage_records')
        ->where('service_id', $serviceId)
        ->orderBy('billing_date', 'desc')
        ->first();

    // Calculate based on config (placeholder for actual monitoring data)
    return $usage;
}

/**
 * Run affiliate commission processing
 */
function runAffiliateCommissionProcessing(array $config): void
{
    if (empty($config['auto_affiliate_processing'])) {
        return;
    }

    $pendingCommissions = Capsule::table('mod_affiliate_commissions')
        ->where('status', 'pending')
        ->where('created_at', '<', date('Y-m-d H:i:s', strtotime('-30 days')))
        ->get();

    foreach ($pendingCommissions as $commission) {
        Capsule::table('mod_affiliate_commissions')
            ->where('id', $commission->id)
            ->update(['status' => 'approved']);
    }
}

/**
 * Run usage report generation
 */
function runUsageReportGeneration(array $config): void
{
    if (empty($config['daily_usage_reports'])) {
        return;
    }

    $yesterday = date('Y-m-d', strtotime('-1 day'));
    
    $usageSummary = Capsule::table('mod_usage_records')
        ->where('billing_date', $yesterday)
        ->selectRaw('SUM(bandwidth_gb) as total_bandwidth, SUM(calculated_amount) as total_cost')
        ->first();

    Capsule::table('mod_daily_usage_reports')->insert([
        'report_date' => $yesterday,
        'total_bandwidth_gb' => $usageSummary->total_bandwidth ?? 0,
        'total_cost' => $usageSummary->total_cost ?? 0,
        'services_count' => Capsule::table('mod_usage_records')
            ->where('billing_date', $yesterday)
            ->count(),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Run cleanup tasks
 */
function runCleanupTasks(array $config): void
{
    $retentionDays = $config['log_retention_days'] ?? 90;
    $cutoffDate = date('Y-m-d H:i:s', strtotime('-' . $retentionDays . ' days'));

    // Clean old execution logs
    Capsule::table('mod_cron_executions')
        ->where('created_at', '<', $cutoffDate)
        ->where('status', 'completed')
        ->delete();

    // Clean old notification logs
    Capsule::table('mod_renewal_notifications')
        ->where('notified_at', '<', date('Y-m-01'))
        ->delete();
}

/**
 * Run statistics update
 */
function runStatisticsUpdate(array $config): void
{
    $today = date('Y-m-d');

    // Update daily statistics
    $newClients = Capsule::table('tblclients')
        ->whereDate('datecreated', $today)
        ->count();

    $newOrders = Capsule::table('tblorders')
        ->whereDate('date', $today)
        ->count();

    $totalRevenue = Capsule::table('tblaccounts')
        ->whereDate('date', $today)
        ->sum('amountin');

    Capsule::table('mod_daily_statistics')->insert([
        'stat_date' => $today,
        'new_clients' => $newClients,
        'new_orders' => $newOrders,
        'total_revenue' => $totalRevenue ?? 0,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Run ticket SLA checks (hourly)
 */
function runTicketSLAChecks(array $config): void
{
    $now = date('Y-m-d H:i:s');

    $slaBreaches = Capsule::table('mod_ticket_sla')
        ->where('status', 'active')
        ->where('first_response_due', '<', $now)
        ->whereNull('first_response_at')
        ->get();

    foreach ($slaBreaches as $breach) {
        Capsule::table('mod_ticket_sla')
            ->where('id', $breach->id)
            ->update(['first_response_status' => 'breached']);

        // Send alert
        sendSLAAlert($breach->ticket_id, 'first_response');
    }
}

/**
 * Run monitoring checks (hourly)
 */
function runMonitoringChecks(array $config): void
{
    $services = Capsule::table('mod_service_monitoring')
        ->where('monitoring_enabled', true)
        ->get();

    foreach ($services as $service) {
        $status = checkServiceHealth($service);
        
        Capsule::table('mod_service_monitoring')
            ->where('id', $service->id)
            ->update([
                'last_check' => date('Y-m-d H:i:s'),
                'status' => $status,
            ]);

        if ($status === 'down') {
            handleServiceDown($service);
        }
    }
}

/**
 * Check service health
 */
function checkServiceHealth(object $service): string
{
    // Placeholder for actual health check
    // In production, this would ping the service URL
    return 'up';
}

/**
 * Handle service down event
 */
function handleServiceDown(object $service): void
{
    Capsule::table('mod_service_incidents')->insert([
        'service_id' => $service->service_id,
        'incident_type' => 'down',
        'started_at' => date('Y-m-d H:i:s'),
        'notified' => false,
    ]);

    // Send notification
    $command = 'SendAdminEmail';
    $postData = [
        'type' => 'service_down',
        'customvars' => base64_encode(json_encode([
            'domain' => $service->domain,
            'service_id' => $service->service_id,
        ])),
    ];
    localAPI($command, $postData);
}

/**
 * Run pending webhook retries (hourly)
 */
function runPendingWebhookRetries(array $config): void
{
    $pending = Capsule::table('mod_webhook_queue')
        ->where('status', 'pending')
        ->where('attempts', '<', 5)
        ->where('next_retry', '<', date('Y-m-d H:i:s'))
        ->get();

    foreach ($pending as $webhook) {
        $result = wp_remote_post($webhook->url, [
            'body' => $webhook->payload,
            'headers' => json_decode($webhook->headers, true) ?? [],
        ]);

        if (!is_wp_error($result)) {
            Capsule::table('mod_webhook_queue')
                ->where('id', $webhook->id)
                ->update(['status' => 'completed']);
        } else {
            Capsule::table('mod_webhook_queue')
                ->where('id', $webhook->id)
                ->update([
                    'attempts' => $webhook->attempts + 1,
                    'next_retry' => date('Y-m-d H:i:s', strtotime('+1 hour')),
                ]);
        }
    }
}

/**
 * Run cache cleanup (hourly)
 */
function runCacheCleanup(array $config): void
{
    $cacheExpiry = $config['cache_ttl'] ?? 3600;
    
    Capsule::table('mod_cache_entries')
        ->where('created_at', '<', date('Y-m-d H:i:s', time() - $cacheExpiry))
        ->delete();
}

/**
 * Send SLA alert
 */
function sendSLAAlert(int $ticketId, string $breachType): void
{
    $command = 'SendAdminEmail';
    $postData = [
        'type' => 'sla_breach',
        'customvars' => base64_encode(json_encode([
            'ticket_id' => $ticketId,
            'breach_type' => $breachType,
        ])),
    ];
    localAPI($command, $postData);
}

// Register hooks
add_hook('DailyCronJob', 1, 'whmcs_cron_automation_daily');
add_hook('HourlyCronJob', 1, 'whmcs_cron_automation_hourly');
```

## Configuration File: config.php

```php
<?php
return [
    // Suspension Settings
    'auto_suspend' => true,
    'suspend_after_days' => 7,

    // Renewal Notifications
    'renewal_notification_days' => [30, 14, 7, 3, 1],
    'expire_warning_days' => 7,

    // Usage Billing
    'usage_billing' => false,
    'daily_usage_reports' => true,

    // Affiliate
    'auto_affiliate_processing' => true,

    // Cleanup
    'log_retention_days' => 90,
    'cache_ttl' => 3600,

    // Monitoring
    'monitoring_enabled' => true,
    'alert_on_down' => true,
];
```

## Database Schema

```php
<?php
use WHMCS\Database\Capsule;

// Cron executions table
if (!Capsule::schema()->hasTable('mod_cron_executions')) {
    Capsule::schema()->create('mod_cron_executions', function ($table) {
        $table->increments('id');
        $table->string('cron_type', 20);
        $table->timestamp('started_at');
        $table->timestamp('completed_at')->nullable();
        $table->string('status', 20);
        $table->timestamp('created_at')->useCurrent();
        
        $table->index(['cron_type', 'created_at']);
    });
}

// Renewal notifications table
if (!Capsule::schema()->hasTable('mod_renewal_notifications')) {
    Capsule::schema()->create('mod_renewal_notifications', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned();
        $table->integer('days_notice');
        $table->timestamp('notified_at');
        
        $table->unique(['service_id', 'days_notice']);
    });
}

// Service expiration log table
if (!Capsule::schema()->hasTable('mod_service_expiration_log')) {
    Capsule::schema()->create('mod_service_expiration_log', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned();
        $table->date('expiry_date');
        $table->integer('days_until_expiry');
        $table->timestamp('checked_at');
        
        $table->index('service_id');
    });
}

// Usage records table
if (!Capsule::schema()->hasTable('mod_usage_records')) {
    Capsule::schema()->create('mod_usage_records', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned();
        $table->date('billing_date');
        $table->longText('usage_data')->nullable();
        $table->decimal('calculated_amount', 10, 2)->default(0);
        $table->timestamp('created_at')->useCurrent();
        
        $table->index(['service_id', 'billing_date']);
    });
}

// Daily usage reports table
if (!Capsule::schema()->hasTable('mod_daily_usage_reports')) {
    Capsule::schema()->create('mod_daily_usage_reports', function ($table) {
        $table->increments('id');
        $table->date('report_date')->unique();
        $table->decimal('total_bandwidth_gb', 10, 2)->default(0);
        $table->decimal('total_cost', 10, 2)->default(0);
        $table->integer('services_count')->default(0);
        $table->timestamp('created_at')->useCurrent();
    });
}

// Daily statistics table
if (!Capsule::schema()->hasTable('mod_daily_statistics')) {
    Capsule::schema()->create('mod_daily_statistics', function ($table) {
        $table->increments('id');
        $table->date('stat_date')->unique();
        $table->integer('new_clients')->default(0);
        $table->integer('new_orders')->default(0);
        $table->decimal('total_revenue', 10, 2)->default(0);
        $table->timestamp('created_at')->useCurrent();
    });
}

// Service incidents table
if (!Capsule::schema()->hasTable('mod_service_incidents')) {
    Capsule::schema()->create('mod_service_incidents', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned();
        $table->string('incident_type', 50);
        $table->timestamp('started_at');
        $table->timestamp('resolved_at')->nullable();
        $table->boolean('notified')->default(false);
    });
}

// Webhook queue table
if (!Capsule::schema()->hasTable('mod_webhook_queue')) {
    Capsule::schema()->create('mod_webhook_queue', function ($table) {
        $table->increments('id');
        $table->string('url', 500);
        $table->longText('payload');
        $table->text('headers')->nullable();
        $table->string('status', 20)->default('pending');
        $table->integer('attempts')->default(0);
        $table->timestamp('next_retry')->nullable();
        $table->timestamp('created_at')->useCurrent();
    });
}

// Cache entries table
if (!Capsule::schema()->hasTable('mod_cache_entries')) {
    Capsule::schema()->create('mod_cache_entries', function ($table) {
        $table->string('cache_key', 100)->primary();
        $table->longText('cache_value')->nullable();
        $table->timestamp('created_at')->useCurrent();
    });
}
```

## Activation & Deactivation

```php
<?php
function whmcs_cron_automation_activate(): array
{
    try {
        require_once __DIR__ . '/schema_migration.php';
        return ['status' => 'success', 'description' => 'Cron Automation Hooks activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_cron_automation_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Cron Automation Hooks deactivated'];
}
```

## Hooks Reference

| Hook | Description |
|------|-------------|
| DailyCronJob | Daily scheduled task execution |
| HourlyCronJob | Hourly scheduled task execution |

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
