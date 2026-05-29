# WHMCS Client Lifecycle Hooks Module

## Overview
This module demonstrates WHMCS hook system implementation for client lifecycle events: ClientAdd, ClientEdit, and ClientDelete.

## Installation
1. Copy module to `/modules/hooks/whmcs_client_lifecycle_hooks/`
2. Activate via WHMCS Admin > System Settings > Module Hooks

## Module File: hooks.php

```php
<?php
/**
 * WHMCS Client Lifecycle Hooks Module
 * 
 * Handles ClientAdd, ClientEdit, ClientDelete hook events
 * for complete client lifecycle management.
 * 
 * @package    WHMCS
 * @subpackage Modules
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 * @license    Commercial License
 * @version    1.0.0
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: ClientAdd
 * Triggered when a new client is created in WHMCS
 * 
 * @param array $params Hook parameters containing client data
 * @return array Modified parameters or empty array
 */
function whmcs_client_lifecycle_hooks_client_add(array $params): array
{
    try {
        // Log the new client creation
        logModuleCall(
            'ClientLifecycleHooks',
            'ClientAdd',
            [
                'userId' => $params['user_id'] ?? null,
                'email' => $params['email'] ?? null,
                'firstName' => $params['firstname'] ?? null,
                'lastName' => $params['lastname'] ?? null,
            ],
            'Client added successfully',
            ''
        );

        // Get module configuration
        $config = require __DIR__ . '/config.php';
        
        // Send welcome email if enabled
        if (!empty($config['send_welcome_email'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $params['user_id'],
                'templatename' => 'Welcome Email',
                'customvars' => base64_encode(json_encode([
                    'client_name' => $params['firstname'] . ' ' . $params['lastname'],
                    'client_email' => $params['email'],
                    'login_url' => $config['client_area_url'] ?? 'https://clients.yourdomain.com/',
                ])),
            ];
            localAPI($command, $postData);
        }

        // Store client metadata in custom table
        $now = date('Y-m-d H:i:s');
        
        Capsule::table('mod_client_lifecycle')->insert([
            'client_id' => $params['user_id'],
            'event_type' => 'created',
            'event_data' => json_encode([
                'source' => $params['creation_source'] ?? 'manual',
                'ip' => $_SERVER['REMOTE_ADDR'] ?? 'unknown',
            ]),
            'created_at' => $now,
            'metadata' => json_encode([
                'hook_version' => '1.0.0',
                'processed_at' => $now,
            ]),
        ]);

        // Sync to external CRM if configured
        if (!empty($config['crm_webhook_url'])) {
            wp_remote_post($config['crm_webhook_url'], [
                'body' => json_encode([
                    'event' => 'client.created',
                    'client_id' => $params['user_id'],
                    'email' => $params['email'],
                    'first_name' => $params['firstname'],
                    'last_name' => $params['lastname'],
                    'company' => $params['companyname'] ?? '',
                    'timestamp' => $now,
                ]),
                'headers' => [
                    'Content-Type' => 'application/json',
                    'X-Webhook-Secret' => $config['crm_webhook_secret'] ?? '',
                ],
            ]);
        }

        // Create audit log entry
        Capsule::table('mod_client_audit_log')->insert([
            'client_id' => $params['user_id'],
            'action' => 'client_add',
            'details' => json_encode($params),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'cli',
            'created_at' => $now,
        ]);

    } catch (\Exception $e) {
        logModuleCall(
            'ClientLifecycleHooks',
            'ClientAdd Error',
            $params,
            $e->getMessage(),
            $e->getTraceAsString()
        );
    }

    return [];
}

/**
 * Hook: ClientEdit
 * Triggered when client profile is modified
 * 
 * @param array $params Hook parameters containing updated client data
 * @return array Modified parameters or empty array
 */
function whmcs_client_lifecycle_hooks_client_edit(array $params): array
{
    try {
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        // Track changes by comparing old vs new values
        $changes = [];
        
        if (isset($params['olddata']) && isset($params['newdata'])) {
            foreach ($params['newdata'] as $key => $newValue) {
                $oldValue = $params['olddata'][$key] ?? null;
                if ($oldValue !== $newValue) {
                    $changes[$key] = [
                        'old' => $oldValue,
                        'new' => $newValue,
                    ];
                }
            }
        }

        // Log significant changes
        $significantFields = ['email', 'firstname', 'lastname', 'companyname', 'address1', 'phonenumber'];
        $significantChanges = array_intersect_key($changes, array_flip($significantFields));

        if (!empty($significantChanges)) {
            logModuleCall(
                'ClientLifecycleHooks',
                'ClientEdit Significant Changes',
                [
                    'client_id' => $params['user_id'],
                    'changes' => $significantChanges,
                ],
                'Significant client data changed',
                ''
            );

            // Send notification for email changes
            if (isset($significantChanges['email'])) {
                // Notify old email address
                if (!empty($config['notify_email_change'])) {
                    $command = 'SendEmail';
                    $postData = [
                        'id' => $params['user_id'],
                        'templatename' => 'Email Change Notification',
                        'customvars' => base64_encode(json_encode([
                            'old_email' => $significantChanges['email']['old'],
                            'new_email' => $significantChanges['email']['new'],
                            'client_name' => $params['firstname'] . ' ' . $params['lastname'],
                        ])),
                    ];
                    localAPI($command, $postData);
                }
            }
        }

        // Store edit history
        Capsule::table('mod_client_lifecycle')->insert([
            'client_id' => $params['user_id'],
            'event_type' => 'edited',
            'event_data' => json_encode([
                'changes' => $changes,
                'source' => $params['admin_user'] ?? 'client',
            ]),
            'created_at' => $now,
            'metadata' => json_encode([
                'hook_version' => '1.0.0',
                'processed_at' => $now,
            ]),
        ]);

        // Update CRM if configured
        if (!empty($config['crm_webhook_url']) && !empty($significantChanges)) {
            wp_remote_post($config['crm_webhook_url'], [
                'body' => json_encode([
                    'event' => 'client.updated',
                    'client_id' => $params['user_id'],
                    'changes' => $significantChanges,
                    'timestamp' => $now,
                ]),
                'headers' => [
                    'Content-Type' => 'application/json',
                    'X-Webhook-Secret' => $config['crm_webhook_secret'] ?? '',
                ],
            ]);
        }

        // Audit log entry
        Capsule::table('mod_client_audit_log')->insert([
            'client_id' => $params['user_id'],
            'action' => 'client_edit',
            'details' => json_encode([
                'changes' => $changes,
                'admin_user' => $params['admin_user'] ?? null,
            ]),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'cli',
            'created_at' => $now,
        ]);

    } catch (\Exception $e) {
        logModuleCall(
            'ClientLifecycleHooks',
            'ClientEdit Error',
            $params,
            $e->getMessage(),
            $e->getTraceAsString()
        );
    }

    return [];
}

/**
 * Hook: ClientDelete
 * Triggered when a client account is deleted
 * 
 * @param array $params Hook parameters containing client data
 * @return array Modified parameters or empty array
 */
function whmcs_client_lifecycle_hooks_client_delete(array $params): array
{
    try {
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        $clientId = $params['user_id'];

        // Store deletion record BEFORE actual deletion
        Capsule::table('mod_client_lifecycle')->insert([
            'client_id' => $clientId,
            'event_type' => 'deleted',
            'event_data' => json_encode([
                'deleted_at' => $now,
                'reason' => $params['delete_reason'] ?? 'manual',
                'admin_user' => $params['admin_user'] ?? 'system',
            ]),
            'created_at' => $now,
            'metadata' => json_encode([
                'hook_version' => '1.0.0',
                'pre_deletion_snapshot' => true,
            ]),
        ]);

        // Notify external systems
        if (!empty($config['crm_webhook_url'])) {
            wp_remote_post($config['crm_webhook_url'], [
                'body' => json_encode([
                    'event' => 'client.deleted',
                    'client_id' => $clientId,
                    'email' => $params['email'] ?? null,
                    'deleted_at' => $now,
                    'reason' => $params['delete_reason'] ?? 'manual',
                ]),
                'headers' => [
                    'Content-Type' => 'application/json',
                    'X-Webhook-Secret' => $config['crm_webhook_secret'] ?? '',
                ],
            ]);
        }

        // Create GDPR-compliant deletion report
        if (!empty($config['gdpr_compliance'])) {
            Capsule::table('mod_client_deletion_report')->insert([
                'client_id' => $clientId,
                'email' => $params['email'] ?? null,
                'deletion_date' => $now,
                'reason' => $params['delete_reason'] ?? 'manual',
                'data_retained' => json_encode($params['retained_data'] ?? []),
                'admin_user' => $params['admin_user'] ?? 'system',
                'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'cli',
                'created_at' => $now,
            ]);
        }

        // Archive related data before potential cascade deletion
        Capsule::table('mod_client_audit_log')->insert([
            'client_id' => $clientId,
            'action' => 'client_delete',
            'details' => json_encode([
                'reason' => $params['delete_reason'] ?? 'manual',
                'admin_user' => $params['admin_user'] ?? 'system',
                'pre_deletion' => true,
            ]),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'cli',
            'created_at' => $now,
        ]);

        logModuleCall(
            'ClientLifecycleHooks',
            'ClientDelete',
            ['client_id' => $clientId],
            'Client deletion processed',
            ''
        );

    } catch (\Exception $e) {
        logModuleCall(
            'ClientLifecycleHooks',
            'ClientDelete Error',
            $params,
            $e->getMessage(),
            $e->getTraceAsString()
        );
    }

    return [];
}

// Register hooks with WHMCS
add_hook('ClientAdd', 1, 'whmcs_client_lifecycle_hooks_client_add');
add_hook('ClientEdit', 1, 'whmcs_client_lifecycle_hooks_client_edit');
add_hook('ClientDelete', 1, 'whmcs_client_lifecycle_hooks_client_delete');
```

## Configuration File: config.php

```php
<?php
/**
 * WHMCS Client Lifecycle Hooks Configuration
 */

return [
    // Email Settings
    'send_welcome_email' => true,
    'notify_email_change' => true,
    'client_area_url' => 'https://clients.yourdomain.com/',

    // CRM Integration
    'crm_webhook_url' => '',
    'crm_webhook_secret' => '',

    // GDPR Compliance
    'gdpr_compliance' => true,

    // Logging
    'log_level' => 'info', // debug, info, warning, error
    'log_retention_days' => 90,

    // Audit Settings
    'audit_all_changes' => true,
    'audit_significant_fields' => ['email', 'firstname', 'lastname', 'companyname', 'address1', 'phonenumber'],
];
```

## Database Schema

```php
<?php
/**
 * Database Schema for Client Lifecycle Hooks Module
 * Run these migrations on module activation
 */

use WHMCS\Database\Capsule;

// Create lifecycle events table
if (!Capsule::schema()->hasTable('mod_client_lifecycle')) {
    Capsule::schema()->create('mod_client_lifecycle', function ($table) {
        $table->increments('id');
        $table->integer('client_id')->unsigned();
        $table->string('event_type', 50); // created, edited, deleted
        $table->longText('event_data')->nullable();
        $table->timestamp('created_at')->useCurrent();
        $table->text('metadata')->nullable();
        
        $table->index('client_id');
        $table->index('event_type');
        $table->index('created_at');
    });
}

// Create audit log table
if (!Capsule::schema()->hasTable('mod_client_audit_log')) {
    Capsule::schema()->create('mod_client_audit_log', function ($table) {
        $table->increments('id');
        $table->integer('client_id')->unsigned();
        $table->string('action', 100);
        $table->longText('details')->nullable();
        $table->string('ip_address', 45)->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('client_id');
        $table->index('action');
        $table->index('created_at');
    });
}

// Create deletion report table for GDPR compliance
if (!Capsule::schema()->hasTable('mod_client_deletion_report')) {
    Capsule::schema()->create('mod_client_deletion_report', function ($table) {
        $table->increments('id');
        $table->integer('client_id')->unsigned();
        $table->string('email', 255)->nullable();
        $table->timestamp('deletion_date');
        $table->string('reason', 100)->nullable();
        $table->longText('data_retained')->nullable();
        $table->string('admin_user', 100)->nullable();
        $table->string('ip_address', 45)->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('client_id');
        $table->index('deletion_date');
    });
}
```

## Activation Function

```php
<?php
/**
 * Module Activation
 */
function whmcs_client_lifecycle_hooks_activate(): array
{
    try {
        // Create database tables
        $schema = file_get_contents(__DIR__ . '/schema.php');
        
        // Run schema creation
        require_once __DIR__ . '/schema_migration.php';

        return [
            'status' => 'success',
            'description' => 'Client Lifecycle Hooks module activated successfully. '
                . 'Handles ClientAdd, ClientEdit, and ClientDelete events.',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to activate module: ' . $e->getMessage(),
        ];
    }
}
```

## Deactivation Function

```php
<?php
/**
 * Module Deactivation
 */
function whmcs_client_lifecycle_hooks_deactivate(): array
{
    try {
        // Optionally clean up data or leave for reactivation
        // Uncomment below to delete data on deactivation:
        // Capsule::schema()->dropIfExists('mod_client_lifecycle');
        // Capsule::schema()->dropIfExists('mod_client_audit_log');
        // Capsule::schema()->dropIfExists('mod_client_deletion_report');

        return [
            'status' => 'success',
            'description' => 'Client Lifecycle Hooks module deactivated.',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to deactivate module: ' . $e->getMessage(),
        ];
    }
}
```

## Schema Migration File: schema_migration.php

```php
<?php
/**
 * Database Schema Migration for Client Lifecycle Hooks
 */

use WHMCS\Database\Capsule;

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Create lifecycle events table
if (!Capsule::schema()->hasTable('mod_client_lifecycle')) {
    Capsule::schema()->create('mod_client_lifecycle', function ($table) {
        $table->increments('id');
        $table->integer('client_id')->unsigned();
        $table->string('event_type', 50);
        $table->longText('event_data')->nullable();
        $table->timestamp('created_at')->useCurrent();
        $table->text('metadata')->nullable();
        
        $table->index('client_id');
        $table->index('event_type');
        $table->index('created_at');
    });
}

// Create audit log table
if (!Capsule::schema()->hasTable('mod_client_audit_log')) {
    Capsule::schema()->create('mod_client_audit_log', function ($table) {
        $table->increments('id');
        $table->integer('client_id')->unsigned();
        $table->string('action', 100);
        $table->longText('details')->nullable();
        $table->string('ip_address', 45)->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('client_id');
        $table->index('action');
        $table->index('created_at');
    });
}

// Create deletion report table
if (!Capsule::schema()->hasTable('mod_client_deletion_report')) {
    Capsule::schema()->create('mod_client_deletion_report', function ($table) {
        $table->increments('id');
        $table->integer('client_id')->unsigned();
        $table->string('email', 255)->nullable();
        $table->timestamp('deletion_date');
        $table->string('reason', 100)->nullable();
        $table->longText('data_retained')->nullable();
        $table->string('admin_user', 100)->nullable();
        $table->string('ip_address', 45)->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('client_id');
        $table->index('deletion_date');
    });
}
```

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2024-01-15 | Initial release |

## Hooks Reference

| Hook Name | Priority | Description |
|-----------|----------|-------------|
| ClientAdd | 1 | Triggered when new client is created |
| ClientEdit | 1 | Triggered when client profile is updated |
| ClientDelete | 1 | Triggered when client account is deleted |

## Requirements

- WHMCS 8.0.0 or higher
- PHP 7.4 or higher
- MySQL 5.7+ or MariaDB 10.2+

## Support

For issues and feature requests, contact: support@hitechcloud.com
