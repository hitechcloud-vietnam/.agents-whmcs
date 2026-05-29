# WHMCS Domain Sync Hooks Module

## Overview
Domain lifecycle management module handling DomainTransferCompleted, DomainRenewed, and other domain-related events.

## Module File: hooks.php

```php
<?php
/**
 * WHMCS Domain Sync Hooks Module
 * 
 * @package    WHMCS
 * @subpackage Modules
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: DomainTransferCompleted
 * Triggered when domain transfer is completed
 */
function whmcs_domain_sync_transfer_completed(array $params): array
{
    try {
        $domainId = $params['domain_id'];
        $domain = $params['domain'];
        $userId = $params['user_id'];
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('DomainSync', 'TransferCompleted', [
            'domain_id' => $domainId,
            'domain' => $domain,
        ], 'Domain transfer completed', '');

        // Update domain status
        Capsule::table('tbldomains')
            ->where('id', $domainId)
            ->update([
                'status' => 'Active',
                'transfer_result' => 'completed',
            ]);

        // Store event
        Capsule::table('mod_domain_events')->insert([
            'domain_id' => $domainId,
            'event_type' => 'transfer_completed',
            'event_data' => json_encode([
                'domain' => $domain,
                'user_id' => $userId,
                'registrar' => $params['registrar'] ?? 'unknown',
            ]),
            'created_at' => $now,
        ]);

        // Update DNS records if configured
        if (!empty($config['auto_configure_dns'])) {
            configureDomainDNS($domainId, $domain);
        }

        // Set up email routing
        if (!empty($config['auto_email_setup'])) {
            setupEmailRouting($domainId, $domain, $userId);
        }

        // Sync with external DNS provider
        if (!empty($config['dns_provider_webhook'])) {
            syncDomainToDNS($domainId, $domain, 'transfer');
        }

        // Send notification
        if (!empty($config['notify_transfer_complete'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $domainId,
                'type' => 'domain',
                'templatename' => 'Domain Transfer Complete',
            ];
            localAPI($command, $postData);
        }

        // Update SSL certificates if applicable
        if (!empty($config['auto_ssl_check'])) {
            checkAndProvisionSSL($domainId, $domain);
        }

    } catch (\Exception $e) {
        logModuleCall('DomainSync', 'TransferCompleted Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: DomainRenewed
 * Triggered when domain is renewed
 */
function whmcs_domain_sync_renewed(array $params): array
{
    try {
        $domainId = $params['domain_id'];
        $domain = $params['domain'];
        $years = $params['years'] ?? 1;
        $expiryDate = $params['next_due_date'];
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('DomainSync', 'DomainRenewed', [
            'domain_id' => $domainId,
            'domain' => $domain,
            'years' => $years,
        ], 'Domain renewed', '');

        // Calculate new expiry
        $newExpiry = date('Y-m-d', strtotime('+' . $years . ' years', strtotime($expiryDate)));

        // Store renewal event
        Capsule::table('mod_domain_events')->insert([
            'domain_id' => $domainId,
            'event_type' => 'renewed',
            'event_data' => json_encode([
                'domain' => $domain,
                'years_added' => $years,
                'previous_expiry' => $expiryDate,
                'new_expiry' => $newExpiry,
                'amount_paid' => $params['amount'] ?? 0,
            ]),
            'created_at' => $now,
        ]);

        // Update renewal tracking
        Capsule::table('mod_domain_renewal_tracking')->insert([
            'domain_id' => $domainId,
            'renewal_date' => $now,
            'years_added' => $years,
            'amount_paid' => $params['amount'] ?? 0,
            'next_expiry' => $newExpiry,
            'created_at' => $now,
        ]);

        // Sync with DNS provider
        if (!empty($config['dns_provider_webhook'])) {
            syncDomainToDNS($domainId, $domain, 'renew');
        }

        // Update SSL certificates
        if (!empty($config['auto_ssl_renew'])) {
            triggerSSLRenewal($domainId, $domain);
        }

        // Send renewal confirmation
        if (!empty($config['notify_renewal'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $domainId,
                'type' => 'domain',
                'templatename' => 'Domain Renewal Confirmation',
            ];
            localAPI($command, $postData);
        }

        // Update WHOIS privacy
        if (!empty($config['maintain_privacy'])) {
            maintainDomainPrivacy($domainId, $domain);
        }

    } catch (\Exception $e) {
        logModuleCall('DomainSync', 'DomainRenewed Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: DomainTransferFailed
 * Triggered when domain transfer fails
 */
function whmcs_domain_sync_transfer_failed(array $params): array
{
    try {
        $domainId = $params['domain_id'] ?? 0;
        $domain = $params['domain'] ?? 'unknown';
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('DomainSync', 'TransferFailed', [
            'domain_id' => $domainId,
            'domain' => $domain,
        ], 'Domain transfer failed', '');

        // Store failure event
        Capsule::table('mod_domain_events')->insert([
            'domain_id' => $domainId,
            'event_type' => 'transfer_failed',
            'event_data' => json_encode([
                'domain' => $domain,
                'failure_reason' => $params['reason'] ?? 'Unknown',
                'user_id' => $params['user_id'] ?? null,
            ]),
            'created_at' => $now,
        ]);

        // Send admin alert
        if (!empty($config['alert_on_failure'])) {
            sendAdminAlert('domain_transfer_failed', [
                'domain' => $domain,
                'reason' => $params['reason'] ?? 'Unknown',
            ]);
        }

    } catch (\Exception $e) {
        logModuleCall('DomainSync', 'TransferFailed Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: DomainExpiryNotification
 * Triggered before domain expiration
 */
function whmcs_domain_sync_expiry_notification(array $params): array
{
    try {
        $domainId = $params['domain_id'];
        $domain = $params['domain'];
        $daysUntilExpiry = $params['days_until_expiry'];
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        // Check if we should send custom notification
        $lastNotification = Capsule::table('mod_domain_notification_log')
            ->where('domain_id', $domainId)
            ->where('notification_type', 'expiry_' . $daysUntilExpiry)
            ->first();

        if ($lastNotification) {
            return [];
        }

        // Store notification sent
        Capsule::table('mod_domain_notification_log')->insert([
            'domain_id' => $domainId,
            'notification_type' => 'expiry_' . $daysUntilExpiry,
            'sent_at' => $now,
        ]);

        // Log event
        Capsule::table('mod_domain_events')->insert([
            'domain_id' => $domainId,
            'event_type' => 'expiry_notification',
            'event_data' => json_encode([
                'domain' => $domain,
                'days_until_expiry' => $daysUntilExpiry,
            ]),
            'created_at' => $now,
        ]);

        // Trigger custom webhook for specific day thresholds
        if (in_array($daysUntilExpiry, [30, 14, 7, 3, 1])) {
            if (!empty($config['expiry_webhook_url'])) {
                wp_remote_post($config['expiry_webhook_url'], [
                    'body' => json_encode([
                        'event' => 'domain.expiry_warning',
                        'domain' => $domain,
                        'days_until_expiry' => $daysUntilExpiry,
                        'timestamp' => $now,
                    ]),
                    'headers' => ['Content-Type' => 'application/json'],
                ]);
            }
        }

    } catch (\Exception $e) {
        logModuleCall('DomainSync', 'ExpiryNotification Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Configure DNS for domain
 */
function configureDomainDNS(int $domainId, string $domain): void
{
    $dnsRecords = [
        ['type' => 'A', 'name' => '@', 'value' => '192.0.2.1', 'ttl' => 3600],
        ['type' => 'CNAME', 'name' => 'www', 'value' => '@', 'ttl' => 3600],
        ['type' => 'MX', 'name' => '@', 'value' => 'mail.' . $domain, 'priority' => 10, 'ttl' => 3600],
    ];

    foreach ($dnsRecords as $record) {
        Capsule::table('mod_domain_dns_records')->insert([
            'domain_id' => $domainId,
            'record_type' => $record['type'],
            'name' => $record['name'],
            'value' => $record['value'],
            'priority' => $record['priority'] ?? null,
            'ttl' => $record['ttl'],
            'is_configured' => true,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * Setup email routing
 */
function setupEmailRouting(int $domainId, string $domain, int $userId): void
{
    Capsule::table('mod_domain_email_routing')->insert([
        'domain_id' => $domainId,
        'user_id' => $userId,
        'domain' => $domain,
        'mx_record' => 'mail.' . $domain,
        'spam_filter' => true,
        'catch_all' => false,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Sync domain to external DNS provider
 */
function syncDomainToDNS(int $domainId, string $domain, string $action): void
{
    $config = require __DIR__ . '/config.php';
    
    wp_remote_post($config['dns_provider_webhook'], [
        'body' => json_encode([
            'event' => 'domain.' . $action,
            'domain_id' => $domainId,
            'domain' => $domain,
            'action' => $action,
            'timestamp' => date('Y-m-d H:i:s'),
        ]),
        'headers' => [
            'Content-Type' => 'application/json',
            'X-Webhook-Secret' => $config['webhook_secret'] ?? '',
        ],
    ]);
}

/**
 * Check and provision SSL
 */
function checkAndProvisionSSL(int $domainId, string $domain): void
{
    Capsule::table('mod_domain_ssl_checks')->insert([
        'domain_id' => $domainId,
        'domain' => $domain,
        'check_type' => 'initial',
        'status' => 'pending',
        'checked_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Trigger SSL renewal
 */
function triggerSSLRenewal(int $domainId, string $domain): void
{
    Capsule::table('mod_domain_ssl_checks')->insert([
        'domain_id' => $domainId,
        'domain' => $domain,
        'check_type' => 'renewal_check',
        'status' => 'pending',
        'checked_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Maintain domain privacy
 */
function maintainDomainPrivacy(int $domainId, string $domain): void
{
    Capsule::table('mod_domain_privacy_maintenance')->insert([
        'domain_id' => $domainId,
        'domain' => $domain,
        'privacy_enabled' => true,
        'maintained_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Send admin alert
 */
function sendAdminAlert(string $type, array $data): void
{
    $command = 'SendAdminEmail';
    $postData = [
        'type' => $type,
        'customvars' => base64_encode(json_encode($data)),
    ];
    localAPI($command, $postData);
}

// Register hooks
add_hook('DomainTransferCompleted', 1, 'whmcs_domain_sync_transfer_completed');
add_hook('DomainRenewed', 1, 'whmcs_domain_sync_renewed');
add_hook('DomainTransferFailed', 1, 'whmcs_domain_sync_transfer_failed');
add_hook('DomainExpiryNotification', 1, 'whmcs_domain_sync_expiry_notification');
```

## Configuration File: config.php

```php
<?php
return [
    // Automation
    'auto_configure_dns' => false,
    'auto_email_setup' => false,
    'maintain_privacy' => true,

    // SSL
    'auto_ssl_check' => true,
    'auto_ssl_renew' => true,

    // Notifications
    'notify_transfer_complete' => true,
    'notify_renewal' => true,
    'alert_on_failure' => true,

    // Webhooks
    'dns_provider_webhook' => '',
    'expiry_webhook_url' => '',
    'webhook_secret' => '',

    // Expiry Notifications
    'expiry_notification_days' => [30, 14, 7, 3, 1],

    // Logging
    'log_level' => 'info',
];
```

## Database Schema

```php
<?php
use WHMCS\Database\Capsule;

// Domain events table
if (!Capsule::schema()->hasTable('mod_domain_events')) {
    Capsule::schema()->create('mod_domain_events', function ($table) {
        $table->increments('id');
        $table->integer('domain_id')->unsigned();
        $table->string('event_type', 50);
        $table->longText('event_data')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('domain_id');
        $table->index('event_type');
    });
}

// Renewal tracking table
if (!Capsule::schema()->hasTable('mod_domain_renewal_tracking')) {
    Capsule::schema()->create('mod_domain_renewal_tracking', function ($table) {
        $table->increments('id');
        $table->integer('domain_id')->unsigned();
        $table->timestamp('renewal_date');
        $table->integer('years_added');
        $table->decimal('amount_paid', 10, 2);
        $table->date('next_expiry');
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('domain_id');
        $table->index('next_expiry');
    });
}

// DNS records table
if (!Capsule::schema()->hasTable('mod_domain_dns_records')) {
    Capsule::schema()->create('mod_domain_dns_records', function ($table) {
        $table->increments('id');
        $table->integer('domain_id')->unsigned();
        $table->string('record_type', 10);
        $table->string('name', 255);
        $table->string('value', 255);
        $table->integer('priority')->nullable();
        $table->integer('ttl')->default(3600);
        $table->boolean('is_configured')->default(false);
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('domain_id');
    });
}

// Email routing table
if (!Capsule::schema()->hasTable('mod_domain_email_routing')) {
    Capsule::schema()->create('mod_domain_email_routing', function ($table) {
        $table->increments('id');
        $table->integer('domain_id')->unsigned();
        $table->integer('user_id')->unsigned();
        $table->string('domain', 255);
        $table->string('mx_record', 255);
        $table->boolean('spam_filter')->default(true);
        $table->boolean('catch_all')->default(false);
        $table->timestamp('created_at')->useCurrent();
    });
}

// Notification log table
if (!Capsule::schema()->hasTable('mod_domain_notification_log')) {
    Capsule::schema()->create('mod_domain_notification_log', function ($table) {
        $table->increments('id');
        $table->integer('domain_id')->unsigned();
        $table->string('notification_type', 50);
        $table->timestamp('sent_at');
    });
}

// SSL checks table
if (!Capsule::schema()->hasTable('mod_domain_ssl_checks')) {
    Capsule::schema()->create('mod_domain_ssl_checks', function ($table) {
        $table->increments('id');
        $table->integer('domain_id')->unsigned();
        $table->string('domain', 255);
        $table->string('check_type', 50);
        $table->string('status', 20);
        $table->timestamp('checked_at');
    });
}

// Privacy maintenance table
if (!Capsule::schema()->hasTable('mod_domain_privacy_maintenance')) {
    Capsule::schema()->create('mod_domain_privacy_maintenance', function ($table) {
        $table->increments('id');
        $table->integer('domain_id')->unsigned();
        $table->string('domain', 255);
        $table->boolean('privacy_enabled');
        $table->timestamp('maintained_at');
    });
}
```

## Activation & Deactivation

```php
<?php
function whmcs_domain_sync_activate(): array
{
    try {
        require_once __DIR__ . '/schema_migration.php';
        return ['status' => 'success', 'description' => 'Domain Sync Hooks activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_domain_sync_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Domain Sync Hooks deactivated'];
}
```

## Hooks Reference

| Hook | Description |
|------|-------------|
| DomainTransferCompleted | Domain transfer finished |
| DomainRenewed | Domain renewal processed |
| DomainTransferFailed | Domain transfer failed |
| DomainExpiryNotification | Expiry warning sent |

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
