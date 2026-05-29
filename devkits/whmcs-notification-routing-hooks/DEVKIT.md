# WHMCS Notification Routing Hooks Module

## Overview
Custom notification routing module with intelligent notification distribution based on rules and conditions.

## Module File: hooks.php

```php
<?php
/**
 * WHMCS Notification Routing Hooks Module
 * 
 * @package    WHMCS
 * @subpackage Modules
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: NotificationPreSend
 * Modify notification before sending
 */
function whmcs_notification_routing_pre_send(array $params): array
{
    try {
        $notification = $params['notification'];
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('NotificationRouting', 'PreSend', [
            'type' => $notification['type'] ?? 'unknown',
        ], 'Processing notification', '');

        // Store notification record
        $notificationId = Capsule::table('mod_notifications')->insertGetId([
            'notification_type' => $notification['type'] ?? 'unknown',
            'notification_name' => $notification['name'] ?? '',
            'recipient' => json_encode($notification['recipients'] ?? []),
            'subject' => $notification['subject'] ?? '',
            'body' => $notification['body'] ?? '',
            'priority' => $notification['priority'] ?? 'normal',
            'status' => 'pending',
            'created_at' => $now,
        ]);

        // Apply routing rules
        $routingResult = applyRoutingRules($notification, $config);
        
        if (!$routingResult['should_send']) {
            Capsule::table('mod_notifications')
                ->where('id', $notificationId)
                ->update(['status' => 'filtered', 'filter_reason' => $routingResult['reason']]);
            
            return ['abort_send' => true];
        }

        // Apply priority escalation
        $notification = applyPriorityEscalation($notification, $config);

        // Apply custom template if configured
        if (!empty($config['custom_templates_enabled'])) {
            $notification = applyCustomTemplate($notification, $config);
        }

        // Store routing decision
        Capsule::table('mod_notification_routing')->insert([
            'notification_id' => $notificationId,
            'routing_rules_applied' => json_encode($routingResult['rules_matched']),
            'recipients_modified' => json_encode($routingResult['recipients']),
            'priority_modified' => $routingResult['priority_changed'],
            'routed_at' => $now,
        ]);

        return $notification;

    } catch (\Exception $e) {
        logModuleCall('NotificationRouting', 'PreSend Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: NotificationSent
 * Log successful notification send
 */
function whmcs_notification_routing_sent(array $params): array
{
    try {
        $now = date('Y-m-d H:i:s');

        $notification = Capsule::table('mod_notifications')
            ->where('subject', $params['subject'] ?? '')
            ->where('status', 'pending')
            ->orderBy('created_at', 'desc')
            ->first();

        if ($notification) {
            Capsule::table('mod_notifications')
                ->where('id', $notification->id)
                ->update([
                    'status' => 'sent',
                    'sent_at' => $now,
                ]);

            // Log delivery
            Capsule::table('mod_notification_delivery')->insert([
                'notification_id' => $notification->id,
                'channel' => 'email',
                'status' => 'delivered',
                'delivered_at' => $now,
            ]);
        }

    } catch (\Exception $e) {
        logModuleCall('NotificationRouting', 'Sent Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: NotificationFailed
 * Handle failed notifications
 */
function whmcs_notification_routing_failed(array $params): array
{
    try {
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        $notification = Capsule::table('mod_notifications')
            ->where('subject', $params['subject'] ?? '')
            ->where('status', 'pending')
            ->orderBy('created_at', 'desc')
            ->first();

        if ($notification) {
            Capsule::table('mod_notifications')
                ->where('id', $notification->id)
                ->update([
                    'status' => 'failed',
                    'error_message' => $params['error'] ?? 'Unknown error',
                    'failed_at' => $now,
                ]);

            // Queue for retry
            if (!empty($config['retry_failed'])) {
                Capsule::table('mod_notification_retry_queue')->insert([
                    'notification_id' => $notification->id,
                    'attempts' => 1,
                    'next_retry' => date('Y-m-d H:i:s', strtotime('+1 hour')),
                    'created_at' => $now,
                ]);
            }

            // Alert admin for critical notifications
            if ($notification->priority === 'critical') {
                alertAdminsOfFailure($notification, $params['error'] ?? '');
            }
        }

    } catch (\Exception $e) {
        logModuleCall('NotificationRouting', 'Failed Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Apply routing rules to notification
 */
function applyRoutingRules(array $notification, array $config): array
{
    $result = [
        'should_send' => true,
        'reason' => '',
        'rules_matched' => [],
        'recipients' => $notification['recipients'] ?? [],
        'priority_changed' => false,
    ];

    $rules = Capsule::table('mod_notification_rules')
        ->where('active', true)
        ->orderBy('priority', 'desc')
        ->get();

    foreach ($rules as $rule) {
        if (matchesRoutingRule($notification, $rule)) {
            $result['rules_matched'][] = $rule->id;

            switch ($rule->action) {
                case 'block':
                    $result['should_send'] = false;
                    $result['reason'] = 'Blocked by rule: ' . $rule->name;
                    break;

                case 'redirect':
                    $result['recipients'] = array_merge(
                        $result['recipients'],
                        explode(',', $rule->action_value)
                    );
                    break;

                case 'escalate':
                    $result['priority_changed'] = true;
                    $notification['priority'] = 'high';
                    break;

                case 'bypass':
                    $result['recipients'] = array_merge(
                        $result['recipients'],
                        [$config['supervisor_email']]
                    );
                    break;
            }
        }
    }

    return $result;
}

/**
 * Check if notification matches routing rule
 */
function matchesRoutingRule(array $notification, object $rule): bool
{
    $conditions = json_decode($rule->conditions, true) ?? [];
    
    foreach ($conditions as $condition) {
        $field = $condition['field'];
        $operator = $condition['operator'];
        $value = $condition['value'];

        $notificationValue = $notification[$field] ?? '';

        switch ($operator) {
            case 'equals':
                if ($notificationValue !== $value) return false;
                break;
            case 'contains':
                if (stripos($notificationValue, $value) === false) return false;
                break;
            case 'starts_with':
                if (strpos($notificationValue, $value) !== 0) return false;
                break;
            case 'ends_with':
                if (substr($notificationValue, -strlen($value)) !== $value) return false;
                break;
            case 'regex':
                if (!preg_match($value, $notificationValue)) return false;
                break;
            case 'in':
                $values = explode(',', $value);
                if (!in_array($notificationValue, $values)) return false;
                break;
        }
    }

    return true;
}

/**
 * Apply priority escalation based on conditions
 */
function applyPriorityEscalation(array $notification, array $config): array
{
    $escalationRules = $config['escalation_rules'] ?? [];

    foreach ($escalationRules as $rule) {
        $typeMatches = in_array($notification['type'] ?? '', $rule['notification_types'] ?? []);
        $amountMatches = isset($rule['min_amount']) && ($notification['amount'] ?? 0) >= $rule['min_amount'];

        if ($typeMatches && $amountMatches) {
            $notification['priority'] = $rule['escalate_to'];
            break;
        }
    }

    return $notification;
}

/**
 * Apply custom template to notification
 */
function applyCustomTemplate(array $notification, array $config): array
{
    $template = Capsule::table('mod_notification_templates')
        ->where('notification_type', $notification['type'] ?? '')
        ->where('active', true)
        ->first();

    if ($template) {
        $notification['subject'] = replaceTemplateVariables($template->subject, $notification);
        $notification['body'] = replaceTemplateVariables($template->body, $notification);
    }

    return $notification;
}

/**
 * Replace template variables with actual values
 */
function replaceTemplateVariables(string $template, array $data): string
{
    $variables = [
        '{{client_name}}' => $data['client_name'] ?? '',
        '{{invoice_id}}' => $data['invoice_id'] ?? '',
        '{{amount}}' => $data['amount'] ?? '',
        '{{due_date}}' => $data['due_date'] ?? '',
        '{{company_name}}' =>WHMCS\Config\Setting::getValue('CompanyName'),
        '{{date}}' => date('Y-m-d'),
    ];

    return str_replace(array_keys($variables), array_values($variables), $template);
}

/**
 * Alert admins of notification failure
 */
function alertAdminsOfFailure(object $notification, string $error): void
{
    $command = 'SendAdminEmail';
    $postData = [
        'type' => 'notification_failed',
        'customvars' => base64_encode(json_encode([
            'notification_type' => $notification->notification_type,
            'subject' => $notification->subject,
            'error' => $error,
        ])),
    ];
    localAPI($command, $postData);
}

/**
 * Process retry queue
 */
function processRetryQueue(): void
{
    $pending = Capsule::table('mod_notification_retry_queue')
        ->where('next_retry', '<', date('Y-m-d H:i:s'))
        ->where('attempts', '<', 5)
        ->get();

    foreach ($pending as $retry) {
        $notification = Capsule::table('mod_notifications')
            ->where('id', $retry->notification_id)
            ->first();

        if ($notification && $notification->status === 'failed') {
            // Retry sending
            $command = 'SendEmail';
            $postData = [
                'type' => $notification->notification_type,
                'subject' => $notification->subject,
                'message' => $notification->body,
            ];
            $result = localAPI($command, $postData);

            if ($result['result'] === 'success') {
                Capsule::table('mod_notifications')
                    ->where('id', $notification->id)
                    ->update(['status' => 'sent', 'sent_at' => date('Y-m-d H:i:s')]);
                
                Capsule::table('mod_notification_retry_queue')
                    ->where('id', $retry->id)
                    ->delete();
            } else {
                Capsule::table('mod_notification_retry_queue')
                    ->where('id', $retry->id)
                    ->update([
                        'attempts' => $retry->attempts + 1,
                        'next_retry' => date('Y-m-d H:i:s', strtotime('+1 hour')),
                    ]);
            }
        }
    }
}

// Register hooks
add_hook('NotificationPreSend', 1, 'whmcs_notification_routing_pre_send');
add_hook('NotificationSent', 1, 'whmcs_notification_routing_sent');
add_hook('NotificationFailed', 1, 'whmcs_notification_routing_failed');
```

## Configuration File: config.php

```php
<?php
return [
    // Custom Templates
    'custom_templates_enabled' => true,

    // Retry Settings
    'retry_failed' => true,
    'max_retry_attempts' => 5,

    // Escalation
    'escalation_rules' => [
        [
            'notification_types' => ['invoice_overdue_critical'],
            'min_amount' => 1000,
            'escalate_to' => 'critical',
        ],
        [
            'notification_types' => ['service_down'],
            'min_amount' => 0,
            'escalate_to' => 'high',
        ],
    ],

    // Supervision
    'supervisor_email' => 'supervisor@hitechcloud.com',

    // Logging
    'log_level' => 'info',
    'log_all_notifications' => true,
];
```

## Database Schema

```php
<?php
use WHMCS\Database\Capsule;

// Notifications table
if (!Capsule::schema()->hasTable('mod_notifications')) {
    Capsule::schema()->create('mod_notifications', function ($table) {
        $table->increments('id');
        $table->string('notification_type', 100);
        $table->string('notification_name', 255);
        $table->longText('recipient')->nullable();
        $table->string('subject', 500);
        $table->longText('body')->nullable();
        $table->enum('priority', ['low', 'normal', 'high', 'critical'])->default('normal');
        $table->enum('status', ['pending', 'sent', 'failed', 'filtered'])->default('pending');
        $table->text('filter_reason')->nullable();
        $table->text('error_message')->nullable();
        $table->timestamp('sent_at')->nullable();
        $table->timestamp('failed_at')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('notification_type');
        $table->index('status');
    });
}

// Notification routing table
if (!Capsule::schema()->hasTable('mod_notification_routing')) {
    Capsule::schema()->create('mod_notification_routing', function ($table) {
        $table->increments('id');
        $table->integer('notification_id')->unsigned();
        $table->longText('routing_rules_applied')->nullable();
        $table->longText('recipients_modified')->nullable();
        $table->boolean('priority_modified')->default(false);
        $table->timestamp('routed_at')->useCurrent();
        
        $table->index('notification_id');
    });
}

// Notification rules table
if (!Capsule::schema()->hasTable('mod_notification_rules')) {
    Capsule::schema()->create('mod_notification_rules', function ($table) {
        $table->increments('id');
        $table->string('name', 100);
        $table->longText('conditions')->nullable();
        $table->enum('action', ['block', 'redirect', 'escalate', 'bypass']);
        $table->string('action_value', 500);
        $table->integer('priority')->default(0);
        $table->boolean('active')->default(true);
        $table->timestamp('created_at')->useCurrent();
    });
}

// Notification templates table
if (!Capsule::schema()->hasTable('mod_notification_templates')) {
    Capsule::schema()->create('mod_notification_templates', function ($table) {
        $table->increments('id');
        $table->string('notification_type', 100);
        $table->string('name', 100);
        $table->string('subject', 500);
        $table->longText('body')->nullable();
        $table->boolean('active')->default(true);
        $table->timestamp('created_at')->useCurrent();
    });
}

// Notification delivery table
if (!Capsule::schema()->hasTable('mod_notification_delivery')) {
    Capsule::schema()->create('mod_notification_delivery', function ($table) {
        $table->increments('id');
        $table->integer('notification_id')->unsigned();
        $table->string('channel', 50);
        $table->string('status', 20);
        $table->text('error_message')->nullable();
        $table->timestamp('delivered_at')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('notification_id');
    });
}

// Retry queue table
if (!Capsule::schema()->hasTable('mod_notification_retry_queue')) {
    Capsule::schema()->create('mod_notification_retry_queue', function ($table) {
        $table->increments('id');
        $table->integer('notification_id')->unsigned();
        $table->integer('attempts')->default(0);
        $table->timestamp('next_retry');
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('next_retry');
    });
}
```

## Activation & Deactivation

```php
<?php
function whmcs_notification_routing_activate(): array
{
    try {
        require_once __DIR__ . '/schema_migration.php';
        
        // Create default rules
        Capsule::table('mod_notification_rules')->insert([
            [
                'name' => 'Block Test Notifications',
                'conditions' => json_encode([['field' => 'subject', 'operator' => 'contains', 'value' => 'Test']]),
                'action' => 'block',
                'action_value' => '',
                'priority' => 100,
            ],
        ]);
        
        return ['status' => 'success', 'description' => 'Notification Routing Hooks activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_notification_routing_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Notification Routing Hooks deactivated'];
}
```

## Hooks Reference

| Hook | Description |
|------|-------------|
| NotificationPreSend | Modify notification before sending |
| NotificationSent | Log successful send |
| NotificationFailed | Handle failed notifications |

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
