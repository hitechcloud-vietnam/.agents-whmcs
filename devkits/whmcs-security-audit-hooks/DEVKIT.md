# WHMCS Security Audit Hooks Module

## Overview
Security monitoring and audit module with LoginFail and AdminLoginFail hooks for threat detection and prevention.

## Module File: hooks.php

```php
<?php
/**
 * WHMCS Security Audit Hooks Module
 * 
 * @package    WHMCS
 * @subpackage Modules
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: LoginFail
 * Triggered when a client login fails
 */
function whmcs_security_audit_login_fail(array $params): array
{
    try {
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        $email = $params['email'] ?? 'unknown';
        $ipAddress = $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0';
        $userAgent = $_SERVER['HTTP_USER_AGENT'] ?? '';

        logModuleCall('SecurityAudit', 'LoginFail', [
            'email' => $email,
            'ip' => $ipAddress,
        ], 'Client login failed', '');

        // Record failed attempt
        $attemptId = Capsule::table('mod_security_login_attempts')->insertGetId([
            'login_type' => 'client',
            'identifier' => $email,
            'ip_address' => $ipAddress,
            'user_agent' => $userAgent,
            'success' => false,
            'attempted_at' => $now,
        ]);

        // Check for brute force
        $recentAttempts = Capsule::table('mod_security_login_attempts')
            ->where('ip_address', $ipAddress)
            ->where('success', false)
            ->where('attempted_at', '>', date('Y-m-d H:i:s', strtotime('-15 minutes')))
            ->count();

        // Check for credential stuffing
        $emailAttempts = Capsule::table('mod_security_login_attempts')
            ->where('identifier', $email)
            ->where('success', false)
            ->where('attempted_at', '>', date('Y-m-d H:i:s', strtotime('-1 hour')))
            ->count();

        // Block IP if too many failures
        if ($recentAttempts >= ($config['max_attempts_15min'] ?? 10)) {
            blockIPAddress($ipAddress, 'brute_force', $config);
        }

        // Check if this looks like credential stuffing
        if ($emailAttempts >= ($config['max_email_attempts'] ?? 5)) {
            logSecurityEvent('credential_stuffing', [
                'email' => $email,
                'ip' => $ipAddress,
                'attempts' => $emailAttempts,
            ], $config);
        }

        // Geo-blocking check
        if (!empty($config['geo_blocking_enabled'])) {
            $country = getCountryFromIP($ipAddress);
            if (in_array($country, $config['blocked_countries'] ?? [])) {
                logSecurityEvent('geo_blocked', [
                    'email' => $email,
                    'ip' => $ipAddress,
                    'country' => $country,
                ], $config);
            }
        }

        // Send alert for suspicious activity
        if ($recentAttempts >= ($config['alert_threshold'] ?? 5)) {
            sendSecurityAlert('brute_force_attempt', [
                'ip' => $ipAddress,
                'attempts' => $recentAttempts,
                'time_window' => '15 minutes',
            ], $config);
        }

    } catch (\Exception $e) {
        logModuleCall('SecurityAudit', 'LoginFail Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: AdminLoginFail
 * Triggered when an admin login fails
 */
function whmcs_security_audit_admin_login_fail(array $params): array
{
    try {
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        $username = $params['username'] ?? 'unknown';
        $ipAddress = $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0';
        $userAgent = $_SERVER['HTTP_USER_AGENT'] ?? '';

        logModuleCall('SecurityAudit', 'AdminLoginFail', [
            'username' => $username,
            'ip' => $ipAddress,
        ], 'Admin login failed', '');

        // Record failed attempt
        Capsule::table('mod_security_login_attempts')->insert([
            'login_type' => 'admin',
            'identifier' => $username,
            'ip_address' => $ipAddress,
            'user_agent' => $userAgent,
            'success' => false,
            'attempted_at' => $now,
        ]);

        // Admin login failures are more serious
        $recentAttempts = Capsule::table('mod_security_login_attempts')
            ->where('ip_address', $ipAddress)
            ->where('login_type', 'admin')
            ->where('success', false)
            ->where('attempted_at', '>', date('Y-m-d H:i:s', strtotime('-15 minutes')))
            ->count();

        // Immediate alert for admin failures
        if ($recentAttempts >= 3) {
            sendSecurityAlert('admin_brute_force', [
                'username' => $username,
                'ip' => $ipAddress,
                'attempts' => $recentAttempts,
            ], $config);

            // Temporary IP block for admin attacks
            if ($recentAttempts >= ($config['admin_max_attempts'] ?? 5)) {
                blockIPAddress($ipAddress, 'admin_brute_force', $config);
            }
        }

        // Check for known bad actors
        if (isKnownBadActor($ipAddress)) {
            sendSecurityAlert('known_attacker', [
                'username' => $username,
                'ip' => $ipAddress,
                'threat_feed' => 'internal',
            ], $config);
        }

        // Two-factor bypass detection
        if (!empty($params['bypass_2fa']) && $params['bypass_2fa']) {
            logSecurityEvent('2fa_bypass_attempt', [
                'username' => $username,
                'ip' => $ipAddress,
            ], $config);
        }

    } catch (\Exception $e) {
        logModuleCall('SecurityAudit', 'AdminLoginFail Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: AdminLoginSuccess
 * Track successful admin logins
 */
function whmcs_security_audit_admin_login_success(array $params): array
{
    try {
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        $username = $params['username'] ?? 'unknown';
        $ipAddress = $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0';

        // Record successful login
        Capsule::table('mod_security_login_attempts')->insert([
            'login_type' => 'admin',
            'identifier' => $username,
            'ip_address' => $ipAddress,
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'success' => true,
            'attempted_at' => $now,
        ]);

        // Check for suspicious session location
        $lastLogin = Capsule::table('mod_security_admin_sessions')
            ->where('username', $username)
            ->where('success', true)
            ->orderBy('login_at', 'desc')
            ->first();

        if ($lastLogin && $lastLogin->ip_address !== $ipAddress) {
            $geoLast = getGeoFromIP($lastLogin->ip_address);
            $geoCurrent = getGeoFromIP($ipAddress);

            if ($geoLast['country'] !== $geoCurrent['country']) {
                sendSecurityAlert('impossible_travel', [
                    'username' => $username,
                    'last_ip' => $lastLogin->ip_address,
                    'current_ip' => $ipAddress,
                    'last_country' => $geoLast['country'],
                    'current_country' => $geoCurrent['country'],
                ], $config);
            }
        }

        // Update admin session tracking
        Capsule::table('mod_security_admin_sessions')->insert([
            'username' => $username,
            'ip_address' => $ipAddress,
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'login_at' => $now,
            'success' => true,
        ]);

    } catch (\Exception $e) {
        logModuleCall('SecurityAudit', 'AdminLoginSuccess Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: ClientLoginSuccess
 * Track successful client logins
 */
function whmcs_security_audit_client_login_success(array $params): array
{
    try {
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        $userId = $params['user_id'] ?? 0;
        $email = $params['email'] ?? 'unknown';
        $ipAddress = $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0';

        // Record successful login
        Capsule::table('mod_security_login_attempts')->insert([
            'login_type' => 'client',
            'identifier' => $email,
            'user_id' => $userId,
            'ip_address' => $ipAddress,
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'success' => true,
            'attempted_at' => $now,
        ]);

        // Update last login
        Capsule::table('tblclients')
            ->where('id', $userId)
            ->update([
                'lastlogin' => $now,
                'lastip' => $ipAddress,
            ]);

    } catch (\Exception $e) {
        logModuleCall('SecurityAudit', 'ClientLoginSuccess Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Block IP address
 */
function blockIPAddress(string $ip, string $reason, array $config): void
{
    $now = date('Y-m-d H:i:s');
    $blockUntil = date('Y-m-d H:i:s', strtotime('+' . ($config['block_duration_minutes'] ?? 60) . ' minutes'));

    Capsule::table('mod_security_ip_blocks')->insert([
        'ip_address' => $ip,
        'reason' => $reason,
        'blocked_at' => $now,
        'block_until' => $blockUntil,
        'auto_block' => true,
    ]);

    logSecurityEvent('ip_blocked', [
        'ip' => $ip,
        'reason' => $reason,
        'block_until' => $blockUntil,
    ], $config);
}

/**
 * Check if IP is blocked
 */
function isIPBlocked(string $ip): bool
{
    return Capsule::table('mod_security_ip_blocks')
        ->where('ip_address', $ip)
        ->where('block_until', '>', date('Y-m-d H:i:s'))
        ->exists();
}

/**
 * Check if IP is known bad actor
 */
function isKnownBadActor(string $ip): bool
{
    return Capsule::table('mod_security_bad_actors')
        ->where('ip_address', $ip)
        ->where('active', true)
        ->exists();
}

/**
 * Get country from IP
 */
function getCountryFromIP(string $ip): ?string
{
    $geo = getGeoFromIP($ip);
    return $geo['country'] ?? null;
}

/**
 * Get geo information from IP
 */
function getGeoFromIP(string $ip): array
{
    // In production, use a GeoIP service
    // For now, return placeholder data
    return [
        'country' => 'Unknown',
        'city' => 'Unknown',
        'latitude' => 0,
        'longitude' => 0,
    ];
}

/**
 * Log security event
 */
function logSecurityEvent(string $eventType, array $data, array $config): void
{
    $now = date('Y-m-d H:i:s');

    Capsule::table('mod_security_events')->insert([
        'event_type' => $eventType,
        'event_data' => json_encode($data),
        'ip_address' => $data['ip'] ?? $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0',
        'severity' => determineSeverity($eventType),
        'logged_at' => $now,
    ]);

    // Store in threat intelligence feed if enabled
    if (!empty($config['threat_intel_enabled'])) {
        if (in_array($eventType, ['brute_force_attempt', 'credential_stuffing', 'known_attacker'])) {
            addToThreatIntel($data['ip'] ?? '', $eventType);
        }
    }
}

/**
 * Determine event severity
 */
function determineSeverity(string $eventType): string
{
    $severityMap = [
        'brute_force_attempt' => 'high',
        'admin_brute_force' => 'critical',
        'credential_stuffing' => 'high',
        'geo_blocked' => 'medium',
        'impossible_travel' => 'high',
        'known_attacker' => 'critical',
        '2fa_bypass_attempt' => 'critical',
        'ip_blocked' => 'low',
    ];

    return $severityMap[$eventType] ?? 'medium';
}

/**
 * Send security alert
 */
function sendSecurityAlert(string $alertType, array $data, array $config): void
{
    $command = 'SendAdminEmail';
    $postData = [
        'type' => 'security_alert',
        'customvars' => base64_encode(json_encode([
            'alert_type' => $alertType,
            'data' => $data,
            'timestamp' => date('Y-m-d H:i:s'),
        ])),
    ];
    localAPI($command, $postData);

    // Webhook alert
    if (!empty($config['security_webhook_url'])) {
        wp_remote_post($config['security_webhook_url'], [
            'body' => json_encode([
                'event' => 'security.alert',
                'alert_type' => $alertType,
                'data' => $data,
                'timestamp' => date('Y-m-d H:i:s'),
            ]),
            'headers' => [
                'Content-Type' => 'application/json',
                'X-Security-Secret' => $config['webhook_secret'] ?? '',
            ],
        ]);
    }
}

/**
 * Add IP to threat intelligence
 */
function addToThreatIntel(string $ip, string $reason): void
{
    $exists = Capsule::table('mod_security_bad_actors')
        ->where('ip_address', $ip)
        ->exists();

    if (!$exists) {
        Capsule::table('mod_security_bad_actors')->insert([
            'ip_address' => $ip,
            'reason' => $reason,
            'active' => true,
            'added_at' => date('Y-m-d H:i:s'),
            'expires_at' => date('Y-m-d H:i:s', strtotime('+30 days')),
        ]);
    }
}

/**
 * Cleanup old records (called from cron)
 */
function cleanupOldRecords(int $retentionDays = 90): void
{
    $cutoff = date('Y-m-d H:i:s', strtotime('-' . $retentionDays . ' days'));

    Capsule::table('mod_security_events')
        ->where('logged_at', '<', $cutoff)
        ->delete();

    Capsule::table('mod_security_login_attempts')
        ->where('attempted_at', '<', $cutoff)
        ->delete();
}

// Register hooks
add_hook('LoginFail', 1, 'whmcs_security_audit_login_fail');
add_hook('AdminLoginFail', 1, 'whmcs_security_audit_admin_login_fail');
add_hook('AdminLoginSuccess', 1, 'whmcs_security_audit_admin_login_success');
add_hook('ClientLoginSuccess', 1, 'whmcs_security_audit_client_login_success');
```

## Configuration File: config.php

```php
<?php
return [
    // Rate Limiting
    'max_attempts_15min' => 10,
    'max_email_attempts' => 5,
    'admin_max_attempts' => 5,

    // Blocking
    'block_duration_minutes' => 60,
    'auto_block_enabled' => true,

    // Geo Blocking
    'geo_blocking_enabled' => false,
    'blocked_countries' => ['XX', 'YY'],

    // Threat Intelligence
    'threat_intel_enabled' => true,

    // Alerts
    'alert_threshold' => 5,
    'alert_email' => 'security@hitechcloud.com',

    // Webhooks
    'security_webhook_url' => '',
    'webhook_secret' => '',

    // Data Retention
    'retention_days' => 90,

    // Logging
    'log_level' => 'info',
];
```

## Database Schema

```php
<?php
use WHMCS\Database\Capsule;

// Login attempts table
if (!Capsule::schema()->hasTable('mod_security_login_attempts')) {
    Capsule::schema()->create('mod_security_login_attempts', function ($table) {
        $table->increments('id');
        $table->enum('login_type', ['client', 'admin']);
        $table->string('identifier', 255);
        $table->integer('user_id')->unsigned()->nullable();
        $table->string('ip_address', 45);
        $table->string('user_agent', 500)->nullable();
        $table->boolean('success')->default(false);
        $table->timestamp('attempted_at')->useCurrent();
        
        $table->index(['ip_address', 'attempted_at']);
        $table->index(['identifier', 'attempted_at']);
        $table->index(['login_type', 'success', 'attempted_at']);
    });
}

// Security events table
if (!Capsule::schema()->hasTable('mod_security_events')) {
    Capsule::schema()->create('mod_security_events', function ($table) {
        $table->increments('id');
        $table->string('event_type', 100);
        $table->longText('event_data')->nullable();
        $table->string('ip_address', 45);
        $table->enum('severity', ['low', 'medium', 'high', 'critical'])->default('medium');
        $table->boolean('reviewed')->default(false);
        $table->string('reviewed_by', 100)->nullable();
        $table->timestamp('logged_at')->useCurrent();
        
        $table->index('event_type');
        $table->index('severity');
        $table->index(['ip_address', 'logged_at']);
    });
}

// IP blocks table
if (!Capsule::schema()->hasTable('mod_security_ip_blocks')) {
    Capsule::schema()->create('mod_security_ip_blocks', function ($table) {
        $table->increments('id');
        $table->string('ip_address', 45)->unique();
        $table->string('reason', 255);
        $table->timestamp('blocked_at');
        $table->timestamp('block_until');
        $table->boolean('auto_block')->default(false);
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('block_until');
    });
}

// Bad actors table
if (!Capsule::schema()->hasTable('mod_security_bad_actors')) {
    Capsule::schema()->create('mod_security_bad_actors', function ($table) {
        $table->increments('id');
        $table->string('ip_address', 45)->unique();
        $table->string('reason', 255);
        $table->boolean('active')->default(true);
        $table->timestamp('added_at');
        $table->timestamp('expires_at')->nullable();
        $table->timestamp('created_at')->useCurrent();
    });
}

// Admin sessions table
if (!Capsule::schema()->hasTable('mod_security_admin_sessions')) {
    Capsule::schema()->create('mod_security_admin_sessions', function ($table) {
        $table->increments('id');
        $table->string('username', 100);
        $table->string('ip_address', 45);
        $table->string('user_agent', 500)->nullable();
        $table->timestamp('login_at');
        $table->boolean('success')->default(true);
        $table->timestamp('created_at')->useCurrent();
        
        $table->index(['username', 'login_at']);
    });
}
```

## Activation & Deactivation

```php
<?php
function whmcs_security_audit_activate(): array
{
    try {
        require_once __DIR__ . '/schema_migration.php';
        return ['status' => 'success', 'description' => 'Security Audit Hooks activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_security_audit_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Security Audit Hooks deactivated'];
}
```

## Hooks Reference

| Hook | Description |
|------|-------------|
| LoginFail | Client login failure |
| AdminLoginFail | Admin login failure |
| AdminLoginSuccess | Admin login success |
| ClientLoginSuccess | Client login success |

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
