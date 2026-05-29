# WHMCS Service Lifecycle Hooks Module

## Overview
Complete service lifecycle management module handling AfterModuleCreate, AfterModuleSuspend, and AfterModuleTerminate hooks.

## Module File: hooks.php

```php
<?php
/**
 * WHMCS Service Lifecycle Hooks Module
 * 
 * @package    WHMCS
 * @subpackage Modules
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: AfterModuleCreate
 * Triggered after service provisioning is complete
 */
function whmcs_service_lifecycle_after_create(array $params): array
{
    try {
        $serviceId = $params['serviceid'];
        $userId = $params['userid'];
        $module = $params['module'];
        $domain = $params['domain'] ?? '';
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('ServiceLifecycle', 'AfterModuleCreate', [
            'service_id' => $serviceId,
            'module' => $module,
        ], 'Service provisioned', '');

        // Get service details
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();

        // Store provisioning event
        Capsule::table('mod_service_lifecycle')->insert([
            'service_id' => $serviceId,
            'event_type' => 'provisioned',
            'event_data' => json_encode([
                'user_id' => $userId,
                'module' => $module,
                'domain' => $domain,
                'product_id' => $service->packageid ?? null,
                'server_id' => $service->server ?? null,
            ]),
            'created_at' => $now,
        ]);

        // Create service credentials storage
        Capsule::table('mod_service_credentials')->insert([
            'service_id' => $serviceId,
            'username' => $params['username'] ?? '',
            'password_encrypted' => encryptValue($params['password'] ?? ''),
            'control_panel_url' => $params['configoption1'] ?? '',
            'created_at' => $now,
        ]);

        // Create welcome guide
        if (!empty($config['send_welcome_guide'])) {
            sendWelcomeGuide($serviceId, $userId, $module);
        }

        // Set up monitoring
        if (!empty($config['enable_monitoring'])) {
            setupServiceMonitoring($serviceId, $domain, $module);
        }

        // Provision add-ons
        if (!empty($config['auto_provision_addons'])) {
            provisionServiceAddons($serviceId, $service->packageid ?? null);
        }

        // Create backup schedule
        if (!empty($config['auto_backup_setup'])) {
            setupBackupSchedule($serviceId, $module);
        }

        // Sync with billing
        if (!empty($config['sync_webhook_url'])) {
            syncServiceToBilling($serviceId, 'provisioned');
        }

        // Send provisioning notification
        if (!empty($config['notify_provisioning'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $serviceId,
                'type' => 'product',
                'templatename' => 'Service Welcome',
            ];
            localAPI($command, $postData);
        }

    } catch (\Exception $e) {
        logModuleCall('ServiceLifecycle', 'AfterModuleCreate Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: AfterModuleSuspend
 * Triggered after service suspension
 */
function whmcs_service_lifecycle_after_suspend(array $params): array
{
    try {
        $serviceId = $params['serviceid'];
        $userId = $params['userid'];
        $reason = $params['suspendreason'] ?? 'Payment overdue';
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('ServiceLifecycle', 'AfterModuleSuspend', [
            'service_id' => $serviceId,
            'reason' => $reason,
        ], 'Service suspended', '');

        // Get previous status
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first(['domainstatus']);

        // Store suspension event
        Capsule::table('mod_service_lifecycle')->insert([
            'service_id' => $serviceId,
            'event_type' => 'suspended',
            'event_data' => json_encode([
                'user_id' => $userId,
                'reason' => $reason,
                'previous_status' => $service->domainstatus ?? 'Unknown',
            ]),
            'created_at' => $now,
        ]);

        // Update suspension tracking
        Capsule::table('mod_service_suspension_tracking')->insert([
            'service_id' => $serviceId,
            'suspension_date' => $now,
            'reason' => $reason,
            'auto_suspend' => (strpos($reason, 'Overdue') !== false),
            'created_at' => $now,
        ]);

        // Stop monitoring alerts
        if (!empty($config['pause_monitoring_on_suspend'])) {
            pauseServiceMonitoring($serviceId);
        }

        // Send suspension notice
        if (!empty($config['notify_suspension'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $serviceId,
                'type' => 'product',
                'templatename' => 'Service Suspended',
                'customvars' => base64_encode(json_encode([
                    'suspension_reason' => $reason,
                ])),
            ];
            localAPI($command, $postData);
        }

        // Log for audit
        Capsule::table('mod_service_audit_log')->insert([
            'service_id' => $serviceId,
            'action' => 'suspend',
            'details' => json_encode(['reason' => $reason]),
            'performed_by' => $params['admin_user'] ?? 'system',
            'created_at' => $now,
        ]);

    } catch (\Exception $e) {
        logModuleCall('ServiceLifecycle', 'AfterModuleSuspend Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: AfterModuleTerminate
 * Triggered after service termination
 */
function whmcs_service_lifecycle_after_terminate(array $params): array
{
    try {
        $serviceId = $params['serviceid'];
        $userId = $params['userid'];
        $reason = $params['terminatereason'] ?? 'Manual';
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('ServiceLifecycle', 'AfterModuleTerminate', [
            'service_id' => $serviceId,
            'reason' => $reason,
        ], 'Service terminated', '');

        // Get service data before termination
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();

        // Store termination event
        Capsule::table('mod_service_lifecycle')->insert([
            'service_id' => $serviceId,
            'event_type' => 'terminated',
            'event_data' => json_encode([
                'user_id' => $userId,
                'reason' => $reason,
                'product_id' => $service->packageid ?? null,
                'domain' => $service->domain ?? '',
                'terminated_by' => $params['admin_user'] ?? 'system',
            ]),
            'created_at' => $now,
        ]);

        // Create data backup before cleanup
        if (!empty($config['backup_before_terminate'])) {
            createTerminationBackup($serviceId, $service);
        }

        // Clean up monitoring
        if (!empty($config['cleanup_monitoring'])) {
            removeServiceMonitoring($serviceId);
        }

        // Send termination notification
        if (!empty($config['notify_termination'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $serviceId,
                'type' => 'product',
                'templatename' => 'Service Terminated',
            ];
            localAPI($command, $postData);
        }

        // Send final data export if requested
        if (!empty($config['data_export_on_terminate'])) {
            requestDataExport($serviceId, $userId);
        }

        // Log for audit
        Capsule::table('mod_service_audit_log')->insert([
            'service_id' => $serviceId,
            'action' => 'terminate',
            'details' => json_encode([
                'reason' => $reason,
                'terminated_by' => $params['admin_user'] ?? 'system',
            ]),
            'performed_by' => $params['admin_user'] ?? 'system',
            'created_at' => $now,
        ]);

        // Archive service credentials
        archiveServiceCredentials($serviceId);

    } catch (\Exception $e) {
        logModuleCall('ServiceLifecycle', 'AfterModuleTerminate Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: AfterModuleUnsuspend
 * Triggered after service unsuspension
 */
function whmcs_service_lifecycle_after_unsuspend(array $params): array
{
    try {
        $serviceId = $params['serviceid'];
        $userId = $params['userid'];
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('ServiceLifecycle', 'AfterModuleUnsuspend', [
            'service_id' => $serviceId,
        ], 'Service unsuspended', '');

        // Calculate suspension duration
        $suspension = Capsule::table('mod_service_suspension_tracking')
            ->where('service_id', $serviceId)
            ->whereNull('unsuspend_date')
            ->first();

        $duration = null;
        if ($suspension) {
            $suspendDate = new DateTime($suspension->suspension_date);
            $unsuspendDate = new DateTime($now);
            $duration = $unsuspendDate->diff($suspendDate)->days;

            Capsule::table('mod_service_suspension_tracking')
                ->where('id', $suspension->id)
                ->update([
                    'unsuspend_date' => $now,
                    'suspension_days' => $duration,
                ]);
        }

        // Store unsuspension event
        Capsule::table('mod_service_lifecycle')->insert([
            'service_id' => $serviceId,
            'event_type' => 'unsuspended',
            'event_data' => json_encode([
                'user_id' => $userId,
                'suspension_duration_days' => $duration,
            ]),
            'created_at' => $now,
        ]);

        // Resume monitoring
        if (!empty($config['enable_monitoring'])) {
            resumeServiceMonitoring($serviceId);
        }

        // Send unsuspension notification
        if (!empty($config['notify_unsuspension'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $serviceId,
                'type' => 'product',
                'templatename' => 'Service Reactivated',
            ];
            localAPI($command, $postData);
        }

    } catch (\Exception $e) {
        logModuleCall('ServiceLifecycle', 'AfterModuleUnsuspend Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Send welcome guide
 */
function sendWelcomeGuide(int $serviceId, int $userId, string $module): void
{
    $guideTemplate = 'Welcome_Guide_' . ucfirst($module);
    
    $command = 'SendEmail';
    $postData = [
        'id' => $serviceId,
        'type' => 'product',
        'templatename' => $guideTemplate,
    ];
    localAPI($command, $postData);
}

/**
 * Setup service monitoring
 */
function setupServiceMonitoring(int $serviceId, string $domain, string $module): void
{
    Capsule::table('mod_service_monitoring')->insert([
        'service_id' => $serviceId,
        'domain' => $domain,
        'module' => $module,
        'monitoring_enabled' => true,
        'check_interval' => 5,
        'last_check' => null,
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Provision service addons
 */
function provisionServiceAddons(int $serviceId, ?int $productId): void
{
    if (!$productId) {
        return;
    }

    $autoAddons = Capsule::table('mod_product_addons')
        ->where('product_id', $productId)
        ->where('auto_provision', true)
        ->get();

    foreach ($autoAddons as $addon) {
        Capsule::table('mod_service_addons')->insert([
            'service_id' => $serviceId,
            'addon_id' => $addon->addon_id,
            'addon_name' => $addon->name,
            'provisioned_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * Setup backup schedule
 */
function setupBackupSchedule(int $serviceId, string $module): void
{
    Capsule::table('mod_service_backups')->insert([
        'service_id' => $serviceId,
        'module' => $module,
        'schedule' => 'daily',
        'retention_days' => 7,
        'enabled' => true,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Sync service to billing
 */
function syncServiceToBilling(int $serviceId, string $action): void
{
    $config = require __DIR__ . '/config.php';
    
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();

    wp_remote_post($config['sync_webhook_url'], [
        'body' => json_encode([
            'event' => 'service.' . $action,
            'service_id' => $serviceId,
            'domain' => $service->domain ?? '',
            'timestamp' => date('Y-m-d H:i:s'),
        ]),
        'headers' => [
            'Content-Type' => 'application/json',
            'X-Webhook-Secret' => $config['webhook_secret'] ?? '',
        ],
    ]);
}

/**
 * Pause service monitoring
 */
function pauseServiceMonitoring(int $serviceId): void
{
    Capsule::table('mod_service_monitoring')
        ->where('service_id', $serviceId)
        ->update([
            'monitoring_enabled' => false,
            'paused_at' => date('Y-m-d H:i:s'),
        ]);
}

/**
 * Resume service monitoring
 */
function resumeServiceMonitoring(int $serviceId): void
{
    Capsule::table('mod_service_monitoring')
        ->where('service_id', $serviceId)
        ->update([
            'monitoring_enabled' => true,
            'resumed_at' => date('Y-m-d H:i:s'),
        ]);
}

/**
 * Remove service monitoring
 */
function removeServiceMonitoring(int $serviceId): void
{
    Capsule::table('mod_service_monitoring')
        ->where('service_id', $serviceId)
        ->update([
            'monitoring_enabled' => false,
            'removed_at' => date('Y-m-d H:i:s'),
        ]);
}

/**
 * Create termination backup
 */
function createTerminationBackup(int $serviceId, object $service): void
{
    Capsule::table('mod_service_backups')->insert([
        'service_id' => $serviceId,
        'backup_type' => 'pre_termination',
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Request data export
 */
function requestDataExport(int $serviceId, int $userId): void
{
    Capsule::table('mod_service_data_export')->insert([
        'service_id' => $serviceId,
        'user_id' => $userId,
        'status' => 'requested',
        'requested_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Archive service credentials
 */
function archiveServiceCredentials(int $serviceId): void
{
    $creds = Capsule::table('mod_service_credentials')
        ->where('service_id', $serviceId)
        ->first();

    if ($creds) {
        Capsule::table('mod_service_credentials_archive')->insert([
            'original_service_id' => $serviceId,
            'username' => $creds->username,
            'password_encrypted' => $creds->password_encrypted,
            'archived_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * Encrypt value for storage
 */
function encryptValue(string $value): string
{
    if (empty($value)) {
        return '';
    }
    return encrypt($value);
}

// Register hooks
add_hook('AfterModuleCreate', 1, 'whmcs_service_lifecycle_after_create');
add_hook('AfterModuleSuspend', 1, 'whmcs_service_lifecycle_after_suspend');
add_hook('AfterModuleTerminate', 1, 'whmcs_service_lifecycle_after_terminate');
add_hook('AfterModuleUnsuspend', 1, 'whmcs_service_lifecycle_after_unsuspend');
```

## Configuration File: config.php

```php
<?php
return [
    // Notifications
    'send_welcome_guide' => true,
    'notify_provisioning' => true,
    'notify_suspension' => true,
    'notify_unsuspension' => true,
    'notify_termination' => true,

    // Automation
    'auto_provision_addons' => false,
    'auto_backup_setup' => true,
    'enable_monitoring' => true,
    'pause_monitoring_on_suspend' => true,

    // Data Management
    'backup_before_terminate' => true,
    'cleanup_monitoring' => true,
    'data_export_on_terminate' => true,

    // Webhooks
    'sync_webhook_url' => '',
    'webhook_secret' => '',

    // Logging
    'log_level' => 'info',
];
```

## Database Schema

```php
<?php
use WHMCS\Database\Capsule;

// Service lifecycle events table
if (!Capsule::schema()->hasTable('mod_service_lifecycle')) {
    Capsule::schema()->create('mod_service_lifecycle', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned();
        $table->string('event_type', 50);
        $table->longText('event_data')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('service_id');
        $table->index('event_type');
    });
}

// Service credentials table
if (!Capsule::schema()->hasTable('mod_service_credentials')) {
    Capsule::schema()->create('mod_service_credentials', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned()->unique();
        $table->string('username', 255);
        $table->text('password_encrypted');
        $table->string('control_panel_url', 500);
        $table->timestamp('created_at')->useCurrent();
    });
}

// Service credentials archive table
if (!Capsule::schema()->hasTable('mod_service_credentials_archive')) {
    Capsule::schema()->create('mod_service_credentials_archive', function ($table) {
        $table->increments('id');
        $table->integer('original_service_id')->unsigned();
        $table->string('username', 255);
        $table->text('password_encrypted');
        $table->timestamp('archived_at')->useCurrent();
    });
}

// Service monitoring table
if (!Capsule::schema()->hasTable('mod_service_monitoring')) {
    Capsule::schema()->create('mod_service_monitoring', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned()->unique();
        $table->string('domain', 255);
        $table->string('module', 100);
        $table->boolean('monitoring_enabled')->default(true);
        $table->integer('check_interval')->default(5);
        $table->timestamp('last_check')->nullable();
        $table->string('status', 20)->default('active');
        $table->timestamp('paused_at')->nullable();
        $table->timestamp('resumed_at')->nullable();
        $table->timestamp('created_at')->useCurrent();
    });
}

// Suspension tracking table
if (!Capsule::schema()->hasTable('mod_service_suspension_tracking')) {
    Capsule::schema()->create('mod_service_suspension_tracking', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned();
        $table->timestamp('suspension_date');
        $table->timestamp('unsuspend_date')->nullable();
        $table->string('reason', 255);
        $table->boolean('auto_suspend')->default(false);
        $table->integer('suspension_days')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('service_id');
    });
}

// Service audit log table
if (!Capsule::schema()->hasTable('mod_service_audit_log')) {
    Capsule::schema()->create('mod_service_audit_log', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned();
        $table->string('action', 50);
        $table->longText('details')->nullable();
        $table->string('performed_by', 100);
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('service_id');
    });
}

// Service backups table
if (!Capsule::schema()->hasTable('mod_service_backups')) {
    Capsule::schema()->create('mod_service_backups', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned();
        $table->string('backup_type', 50);
        $table->string('schedule', 20)->nullable();
        $table->integer('retention_days')->nullable();
        $table->boolean('enabled')->default(true);
        $table->string('status', 20)->default('pending');
        $table->timestamp('last_backup')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('service_id');
    });
}

// Service addons table
if (!Capsule::schema()->hasTable('mod_service_addons')) {
    Capsule::schema()->create('mod_service_addons', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned();
        $table->integer('addon_id')->unsigned();
        $table->string('addon_name', 255);
        $table->timestamp('provisioned_at')->useCurrent();
        
        $table->index('service_id');
    });
}

// Data export requests table
if (!Capsule::schema()->hasTable('mod_service_data_export')) {
    Capsule::schema()->create('mod_service_data_export', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned();
        $table->integer('user_id')->unsigned();
        $table->string('status', 20);
        $table->timestamp('requested_at')->useCurrent();
        $table->timestamp('completed_at')->nullable();
    });
}

// Product addons mapping table
if (!Capsule::schema()->hasTable('mod_product_addons')) {
    Capsule::schema()->create('mod_product_addons', function ($table) {
        $table->increments('id');
        $table->integer('product_id')->unsigned();
        $table->integer('addon_id')->unsigned();
        $table->boolean('auto_provision')->default(false);
    });
}
```

## Activation & Deactivation

```php
<?php
function whmcs_service_lifecycle_activate(): array
{
    try {
        require_once __DIR__ . '/schema_migration.php';
        return ['status' => 'success', 'description' => 'Service Lifecycle Hooks activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_service_lifecycle_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Service Lifecycle Hooks deactivated'];
}
```

## Hooks Reference

| Hook | Description |
|------|-------------|
| AfterModuleCreate | Service provisioned |
| AfterModuleSuspend | Service suspended |
| AfterModuleTerminate | Service terminated |
| AfterModuleUnsuspend | Service reactivated |

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
