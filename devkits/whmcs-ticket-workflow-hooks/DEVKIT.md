# WHMCS Ticket Workflow Hooks Module

## Overview
Complete ticket workflow automation module handling TicketOpen, TicketReply, and TicketClose events with SLA tracking and escalation.

## Module File: hooks.php

```php
<?php
/**
 * WHMCS Ticket Workflow Hooks Module
 * 
 * @package    WHMCS
 * @subpackage Modules
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: TicketOpen
 * Triggered when a new support ticket is opened
 */
function whmcs_ticket_workflow_ticket_open(array $params): array
{
    try {
        $ticketId = $params['ticketid'];
        $userId = $params['userid'];
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('TicketWorkflow', 'TicketOpen', ['ticket_id' => $ticketId], 'New ticket opened', '');

        // Determine priority based on keywords
        $priority = determineTicketPriority($params['subject'], $params['message']);
        
        // Store ticket opening event
        Capsule::table('mod_ticket_events')->insert([
            'ticket_id' => $ticketId,
            'event_type' => 'opened',
            'event_data' => json_encode([
                'user_id' => $userId,
                'subject' => $params['subject'],
                'priority' => $priority,
                'department_id' => $params['deptid'] ?? null,
            ]),
            'created_at' => $now,
        ]);

        // Create SLA record
        $slaDue = calculateSLADueDate($priority, $now);
        Capsule::table('mod_ticket_sla')->insert([
            'ticket_id' => $ticketId,
            'priority' => $priority,
            'first_response_due' => $slaDue['first_response'],
            'resolution_due' => $slaDue['resolution'],
            'status' => 'active',
            'created_at' => $now,
        ]);

        // Auto-assign based on keywords/department
        if (!empty($config['auto_assign_enabled'])) {
            $assignedTo = autoAssignTicket($params['subject'], $params['message'], $params['deptid'] ?? null);
            if ($assignedTo) {
                updateTicketAssignment($ticketId, $assignedTo);
            }
        }

        // Send auto-response
        if (!empty($config['send_auto_response'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $ticketId,
                'type' => 'support',
                'customtemplate' => 'support_ticket_auto_response',
            ];
            localAPI($command, $postData);
        }

        // Escalate high priority tickets
        if ($priority === 'High' || $priority === 'Medium') {
            escalateTicket($ticketId, $params);
        }

        // Notify specific departments
        notifyDepartment($ticketId, $params['deptid'] ?? null);

    } catch (\Exception $e) {
        logModuleCall('TicketWorkflow', 'TicketOpen Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: TicketReply
 * Triggered when a reply is added to a ticket
 */
function whmcs_ticket_workflow_ticket_reply(array $params): array
{
    try {
        $ticketId = $params['ticketid'];
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        $isClientReply = ($params['replytype'] ?? 'user') === 'user';
        
        logModuleCall('TicketWorkflow', 'TicketReply', [
            'ticket_id' => $ticketId,
            'is_client' => $isClientReply,
        ], 'Ticket reply received', '');

        // Store reply event
        Capsule::table('mod_ticket_events')->insert([
            'ticket_id' => $ticketId,
            'event_type' => $isClientReply ? 'client_reply' : 'staff_reply',
            'event_data' => json_encode([
                'reply_id' => $params['replyid'] ?? null,
                'is_client' => $isClientReply,
                'admin_user' => $params['admin_user'] ?? null,
            ]),
            'created_at' => $now,
        ]);

        // Update SLA first response if this is staff reply
        if (!$isClientReply) {
            Capsule::table('mod_ticket_sla')
                ->where('ticket_id', $ticketId)
                ->update([
                    'first_response_at' => $now,
                    'first_response_status' => 'met',
                ]);
        }

        // Detect sentiment and escalate if negative
        if ($isClientReply && !empty($config['sentiment_detection'])) {
            $sentiment = detectSentiment($params['message'] ?? '');
            if ($sentiment === 'negative' && !empty($config['escalate_negative'])) {
                escalateTicket($ticketId, ['reason' => 'negative_sentiment']);
            }
        }

        // Auto-close if marked as such
        if (!$isClientReply && !empty($params['close']) && !empty($config['allow_auto_close'])) {
            $command = 'CloseTicket';
            $postData = ['ticketid' => $ticketId];
            localAPI($command, $postData);
        }

        // Survey request after resolution
        if ($params['status'] === 'Resolved' && !empty($config['send_survey'])) {
            requestSurvey($ticketId, $params['userid']);
        }

    } catch (\Exception $e) {
        logModuleCall('TicketWorkflow', 'TicketReply Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: TicketClose
 * Triggered when a ticket is closed
 */
function whmcs_ticket_workflow_ticket_close(array $params): array
{
    try {
        $ticketId = $params['ticketid'];
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('TicketWorkflow', 'TicketClose', ['ticket_id' => $ticketId], 'Ticket closed', '');

        // Calculate resolution time
        $sla = Capsule::table('mod_ticket_sla')
            ->where('ticket_id', $ticketId)
            ->first();

        $openedAt = Capsule::table('mod_ticket_events')
            ->where('ticket_id', $ticketId)
            ->where('event_type', 'opened')
            ->value('created_at');

        $resolutionTime = $openedAt ? calculateResolutionTime($openedAt, $now) : null;

        // Store closure event
        Capsule::table('mod_ticket_events')->insert([
            'ticket_id' => $ticketId,
            'event_type' => 'closed',
            'event_data' => json_encode([
                'resolution_time_minutes' => $resolutionTime,
                'closed_by' => $params['admin_user'] ?? 'client',
                'close_reason' => $params['close_reason'] ?? null,
            ]),
            'created_at' => $now,
        ]);

        // Update SLA record
        if ($sla) {
            Capsule::table('mod_ticket_sla')
                ->where('ticket_id', $ticketId)
                ->update([
                    'resolved_at' => $now,
                    'resolution_time_minutes' => $resolutionTime,
                    'resolution_status' => $resolutionTime <= $sla->resolution_due ? 'met' : 'breached',
                    'status' => 'closed',
                ]);
        }

        // Send satisfaction survey
        if (!empty($config['send_survey'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $ticketId,
                'type' => 'support',
                'templatename' => 'Support Ticket Survey',
            ];
            localAPI($command, $postData);
        }

        // Update staff statistics
        if (!empty($params['admin_user'])) {
            updateStaffStats($params['admin_user'], $resolutionTime);
        }

    } catch (\Exception $e) {
        logModuleCall('TicketWorkflow', 'TicketClose Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Determine ticket priority based on content
 */
function determineTicketPriority(string $subject, string $message): string
{
    $urgentKeywords = ['urgent', 'emergency', 'critical', 'asap', 'down', 'outage', 'broken', 'not working'];
    $highKeywords = ['important', 'issue', 'problem', 'error', 'failed', 'cannot'];
    
    $content = strtolower($subject . ' ' . $message);
    
    foreach ($urgentKeywords as $keyword) {
        if (strpos($content, $keyword) !== false) {
            return 'High';
        }
    }
    
    foreach ($highKeywords as $keyword) {
        if (strpos($content, $keyword) !== false) {
            return 'Medium';
        }
    }
    
    return 'Low';
}

/**
 * Calculate SLA due dates
 */
function calculateSLADueDate(string $priority, string $startTime): array
{
    $slaHours = [
        'High' => 4,
        'Medium' => 24,
        'Low' => 72,
    ];
    
    $hours = $slaHours[$priority] ?? 72;
    $firstResponse = date('Y-m-d H:i:s', strtotime($startTime) + ($hours * 3600));
    $resolution = date('Y-m-d H:i:s', strtotime($startTime) + ($hours * 4 * 3600));
    
    return [
        'first_response' => $firstResponse,
        'resolution' => $resolution,
    ];
}

/**
 * Auto-assign ticket based on keywords
 */
function autoAssignTicket(string $subject, string $message, ?int $departmentId): ?string
{
    $rules = Capsule::table('mod_ticket_assignment_rules')
        ->get();
    
    foreach ($rules as $rule) {
        $keywords = explode(',', $rule->keywords);
        $content = strtolower($subject . ' ' . $message);
        
        foreach ($keywords as $keyword) {
            if (strpos($content, strtolower(trim($keyword))) !== false) {
                return $rule->assign_to;
            }
        }
    }
    
    return null;
}

/**
 * Update ticket assignment
 */
function updateTicketAssignment(int $ticketId, string $adminUser): void
{
    $command = 'UpdateTicket';
    $postData = [
        'ticketid' => $ticketId,
        'assignedto' => $adminUser,
    ];
    localAPI($command, $postData);
}

/**
 * Escalate ticket
 */
function escalateTicket(int $ticketId, array $params): void
{
    $command = 'UpdateTicket';
    $postData = [
        'ticketid' => $ticketId,
        'priority' => 'High',
        'flag' => 'admin',
    ];
    localAPI($command, $postData);

    Capsule::table('mod_ticket_events')->insert([
        'ticket_id' => $ticketId,
        'event_type' => 'escalated',
        'event_data' => json_encode(['reason' => $params['reason'] ?? 'keyword_match']),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Detect sentiment in message
 */
function detectSentiment(string $message): string
{
    $negativeWords = ['frustrated', 'angry', 'terrible', 'worst', 'disappointed', 'unacceptable', 'ridiculous'];
    $positiveWords = ['thanks', 'thank you', 'great', 'excellent', 'appreciate', 'helpful'];
    
    $content = strtolower($message);
    
    $negativeCount = 0;
    $positiveCount = 0;
    
    foreach ($negativeWords as $word) {
        $negativeCount += substr_count($content, $word);
    }
    
    foreach ($positiveWords as $word) {
        $positiveCount += substr_count($content, $word);
    }
    
    if ($negativeCount > $positiveCount) {
        return 'negative';
    } elseif ($positiveCount > $negativeCount) {
        return 'positive';
    }
    
    return 'neutral';
}

/**
 * Request survey from client
 */
function requestSurvey(int $ticketId, int $userId): void
{
    Capsule::table('mod_ticket_survey_requests')->insert([
        'ticket_id' => $ticketId,
        'user_id' => $userId,
        'requested_at' => date('Y-m-d H:i:s'),
        'status' => 'pending',
    ]);
}

/**
 * Calculate resolution time in minutes
 */
function calculateResolutionTime(string $start, string $end): int
{
    $startTime = new DateTime($start);
    $endTime = new DateTime($end);
    return $startTime->diff($endTime)->i + ($startTime->diff($endTime)->h * 60);
}

/**
 * Update staff statistics
 */
function updateStaffStats(string $adminUser, ?int $resolutionTime): void
{
    Capsule::table('mod_staff_ticket_stats')
        ->where('admin_user', $adminUser)
        ->update([
            'tickets_resolved' => Capsule::raw('tickets_resolved + 1'),
            'avg_resolution_time' => Capsule::raw('(avg_resolution_time * tickets_resolved + ' . ($resolutionTime ?? 0) . ') / (tickets_resolved + 1)'),
            'last_updated' => date('Y-m-d H:i:s'),
        ]);
}

/**
 * Notify department of new ticket
 */
function notifyDepartment(int $ticketId, ?int $departmentId): void
{
    if (!$departmentId) {
        return;
    }
    
    $admins = Capsule::table('tbladmins')
        ->join('tbladminperms', 'tbladmins.id', '=', 'tbladminperms.adminid')
        ->where('tbladminperms.permid', function ($query) use ($departmentId) {
            $query->select('id')
                ->from('tblpermissions')
                ->where('permission', 'department' . $departmentId);
        })
        ->get(['tbladmins.email', 'tbladmins.firstname', 'tbladmins.lastname']);
    
    foreach ($admins as $admin) {
        $command = 'SendAdminEmail';
        $postData = [
            'type' => 'ticket_notification',
            'customvars' => base64_encode(json_encode([
                'admin_name' => $admin->firstname . ' ' . $admin->lastname,
                'ticket_id' => $ticketId,
            ])),
        ];
        localAPI($command, $postData);
    }
}

// Register hooks
add_hook('TicketOpen', 1, 'whmcs_ticket_workflow_ticket_open');
add_hook('TicketReply', 1, 'whmcs_ticket_workflow_ticket_reply');
add_hook('TicketClose', 1, 'whmcs_ticket_workflow_ticket_close');
```

## Configuration File: config.php

```php
<?php
return [
    // Auto-assignment
    'auto_assign_enabled' => true,
    'auto_response_department' => null,

    // Email Notifications
    'send_auto_response' => true,
    'send_survey' => true,

    // SLA Settings
    'sla_first_response_hours' => ['high' => 4, 'medium' => 24, 'low' => 72],
    'sla_resolution_hours' => ['high' => 16, 'medium' => 96, 'low' => 288],

    // Sentiment Detection
    'sentiment_detection' => true,
    'escalate_negative' => true,

    // Auto-close
    'allow_auto_close' => false,
    'auto_close_hours' => 72,

    // Escalation
    'escalate_keywords' => ['urgent', 'emergency', 'critical', 'outage'],

    // Logging
    'log_level' => 'info',
];
```

## Database Schema

```php
<?php
use WHMCS\Database\Capsule;

// Ticket events table
if (!Capsule::schema()->hasTable('mod_ticket_events')) {
    Capsule::schema()->create('mod_ticket_events', function ($table) {
        $table->increments('id');
        $table->integer('ticket_id')->unsigned();
        $table->string('event_type', 50);
        $table->longText('event_data')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('ticket_id');
        $table->index('event_type');
    });
}

// SLA tracking table
if (!Capsule::schema()->hasTable('mod_ticket_sla')) {
    Capsule::schema()->create('mod_ticket_sla', function ($table) {
        $table->increments('id');
        $table->integer('ticket_id')->unsigned()->unique();
        $table->string('priority', 20);
        $table->timestamp('first_response_due')->nullable();
        $table->timestamp('resolution_due')->nullable();
        $table->timestamp('first_response_at')->nullable();
        $table->timestamp('resolved_at')->nullable();
        $table->integer('resolution_time_minutes')->nullable();
        $table->enum('first_response_status', ['met', 'breached', 'pending'])->default('pending');
        $table->enum('resolution_status', ['met', 'breached', 'pending'])->default('pending');
        $table->enum('status', ['active', 'closed'])->default('active');
        $table->timestamp('created_at')->useCurrent();
    });
}

// Assignment rules table
if (!Capsule::schema()->hasTable('mod_ticket_assignment_rules')) {
    Capsule::schema()->create('mod_ticket_assignment_rules', function ($table) {
        $table->increments('id');
        $table->string('name', 100);
        $table->text('keywords');
        $table->string('assign_to', 100);
        $table->integer('priority')->default(0);
        $table->boolean('active')->default(true);
        $table->timestamp('created_at')->useCurrent();
    });
}

// Staff stats table
if (!Capsule::schema()->hasTable('mod_staff_ticket_stats')) {
    Capsule::schema()->create('mod_staff_ticket_stats', function ($table) {
        $table->increments('id');
        $table->string('admin_user', 100)->unique();
        $table->integer('tickets_resolved')->default(0);
        $table->decimal('avg_resolution_time', 10, 2)->default(0);
        $table->timestamp('last_updated')->useCurrent();
    });
}

// Survey requests table
if (!Capsule::schema()->hasTable('mod_ticket_survey_requests')) {
    Capsule::schema()->create('mod_ticket_survey_requests', function ($table) {
        $table->increments('id');
        $table->integer('ticket_id')->unsigned();
        $table->integer('user_id')->unsigned();
        $table->timestamp('requested_at')->useCurrent();
        $table->enum('status', ['pending', 'completed', 'declined'])->default('pending');
        $table->integer('rating')->nullable();
        $table->text('feedback')->nullable();
    });
}
```

## Activation & Deactivation

```php
<?php
function whmcs_ticket_workflow_activate(): array
{
    try {
        require_once __DIR__ . '/schema_migration.php';
        
        // Initialize default assignment rules
        Capsule::table('mod_ticket_assignment_rules')->insert([
            ['name' => 'Billing Issues', 'keywords' => 'billing,invoice,payment,refund', 'assign_to' => 'billing_admin', 'priority' => 10],
            ['name' => 'Technical Support', 'keywords' => 'error,bug,issue,problem,not working', 'assign_to' => 'tech_admin', 'priority' => 10],
            ['name' => 'Sales Inquiries', 'keywords' => 'purchase,order,pricing,quote', 'assign_to' => 'sales_admin', 'priority' => 10],
        ]);
        
        return ['status' => 'success', 'description' => 'Ticket Workflow Hooks activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_ticket_workflow_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Ticket Workflow Hooks deactivated'];
}
```

## Hooks Reference

| Hook | Description |
|------|-------------|
| TicketOpen | New ticket created |
| TicketReply | Reply added to ticket |
| TicketClose | Ticket closed |

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
