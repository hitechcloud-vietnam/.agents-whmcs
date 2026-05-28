# WHMCS Hook Development Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Comprehensive guide for developing custom hooks including advanced patterns, hook priority management, and hook-based automation systems.

## When to Use

- Creating custom automation logic that responds to WHMCS events
- Extending WHMCS functionality without modifying core files
- Building event-driven integrations with external services
- Creating complex multi-hook workflows

## Hook System Overview

### 1. Basic Hook Registration

```php
<?php
// hooks.php - Main hook file
if (!defined("WHMCS")) { die("Direct access denied"); }

// Simple hook
add_hook('ClientAdd', 1, function($vars) {
    // $vars contains hook-specific data
    $clientId = $vars['userid'];
    $email = $vars['email'];

    // Your logic here
});

// Multiple hooks in one file
$hooks = [
    'ClientAdd' => 'handleClientAdd',
    'ClientEdit' => 'handleClientEdit',
    'InvoicePaid' => 'handleInvoicePaid',
];

foreach ($hooks as $hookName => $callback) {
    add_hook($hookName, 1, function($vars) use ($callback) {
        $callback($vars);
    });
}
```

### 2. Hook Priority Guide

| Priority | Range | Use Case |
|----------|-------|----------|
| 0 | System critical | Core operations, early exit prevention |
| 1-5 | Critical processing | Main business logic |
| 10-20 | Standard processing | Default for most hooks |
| 50-100 | Post-processing | Logging, notifications |
| 100-500 | Late processing | Cleanup, archiving |
| 1000+ | Debug/monitoring | Performance tracking |

### 3. Hook Variables Reference

```php
<?php
// Common hook variable structures

// Client Hooks
add_hook('ClientAdd', 1, function($vars) {
    // $vars = [
    //     'userid' => 123,
    //     'firstname' => 'John',
    //     'lastname' => 'Doe',
    //     'email' => 'john@example.com',
    //     'companyname' => '',
    //     'address1' => '123 Main St',
    //     'city' => 'San Francisco',
    //     'state' => 'CA',
    //     'country' => 'US',
    //     'phonenumber' => '+1-555-1234',
    //     'password' => 'hashed_password',
    // ]
});

// Service Hooks
add_hook('AfterModuleCreate', 1, function($vars) {
    // $vars = [
    //     'serviceid' => 789,
    //     'userid' => 123,
    //     'pid' => 1,
    //     'domain' => 'example.com',
    //     'username' => 'user123',
    //     'password' => 'encrypted_password',
    //     'params' => [...], // Module config
    // ]
});

// Invoice Hooks
add_hook('InvoicePaid', 1, function($vars) {
    // $vars = [
    //     'invoiceid' => 456,
    //     'userid' => 123,
    //     'total' => 29.99,
    //     'status' => 'Paid',
    //     'transid' => 'TXN123',
    // ]
});
```

## Advanced Hook Patterns

### 1. Hook Class Pattern (OOP)

```php
<?php
// includes/hooks/AdvancedHooks.php

namespace MyModule\Hooks;

class ClientEventHandler {
    public function handleClientAdd(array $vars): void {
        $this->log("Client added: {$vars['userid']}");
        $this->syncToExternal($vars);
        $this->sendWelcomeEmail($vars);
    }

    public function handleClientEdit(array $vars): void {
        $this->log("Client updated: {$vars['userid']}");
        $this->syncChanges($vars);
    }

    public function handleClientDelete(array $vars): void {
        $this->log("Client deleted: {$vars['userid']}");
        $this->cleanupExternalAccounts($vars['userid']);
    }

    private function log(string $message): void {
        logActivity("[MyModule] " . $message);
    }

    private function syncToExternal(array $vars): void {
        // Sync to external CRM/API
    }

    private function sendWelcomeEmail(array $vars): void {
        // Send custom welcome email
    }

    private function syncChanges(array $vars): void {
        // Sync changes to external system
    }

    private function cleanupExternalAccounts(int $clientId): void {
        // Clean up external accounts
    }
}

// Register hooks
$handler = new \MyModule\Hooks\ClientEventHandler();

add_hook('ClientAdd', 1, function($vars) use ($handler) {
    $handler->handleClientAdd($vars);
});

add_hook('ClientEdit', 1, function($vars) use ($handler) {
    $handler->handleClientEdit($vars);
});

add_hook('ClientDelete', 1, function($vars) use ($handler) {
    $handler->handleClientDelete($vars);
});
```

### 2. Hook Priority Chaining

```php
<?php
// hooks.php - Multiple hooks on same event with different priorities

// Priority 1: Initial validation (early exit possible)
add_hook('NewOrder', 1, function($vars) {
    // Check for fraud indicators
    $fraudScore = checkFraudScore($vars);

    if ($fraudScore > 80) {
        // Can use this to prevent order or mark for review
        return ['abort' => true, 'reason' => 'High fraud score'];
    }
});

// Priority 10: Main processing
add_hook('NewOrder', 10, function($vars) {
    // Standard order processing
    // This won't run if priority 1 returns abort
});

// Priority 50: Post-processing
add_hook('NewOrder', 50, function($vars) {
    // Send notifications
    // Update analytics
    // Sync to external systems
});

// Priority 100: Logging
add_hook('NewOrder', 100, function($vars) {
    // Log order details
    // Send to data warehouse
});
```

### 3. Conditional Hook Execution

```php
<?php
// hooks.php - Conditional hook execution

// Only run for specific products
add_hook('AfterModuleCreate', 1, function($vars) {
    // Get product ID from service
    $service = \WHMCS\Database\Capsule::table('tblhosting')
        ->where('id', $vars['serviceid'])
        ->first();

    // Only process specific products
    $targetProducts = [1, 5, 10]; // VPS products

    if (!in_array($service->pid, $targetProducts)) {
        return; // Skip this hook for other products
    }

    // Do VPS-specific logic
    setupVPSEnvironment($vars);
});

// Only run for specific clients
add_hook('InvoicePaid', 1, function($vars) {
    $client = \WHMCS\Database\Capsule::table('tblclients')
        ->where('id', $vars['userid'])
        ->first();

    // Skip for specific client groups (e.g., resellers)
    if ($client->groupid == 5) {
        return;
    }

    // Standard processing for other clients
    processStandardPayment($vars);
});

// Only run during specific hours
add_hook('DailyCronJob', 1, function($vars) {
    $hour = (int) date('H');

    // Only run between 2 AM and 4 AM
    if ($hour < 2 || $hour >= 4) {
        return;
    }

    // Heavy processing during low-usage hours
    runHeavyMaintenance();
});
```

### 4. Async Hook Processing

```php
<?php
// hooks.php - Queue-based async processing

use WHMCS\Database\Capsule;

// Hook with immediate queueing
add_hook('InvoicePaid', 10, function($vars) {
    // Queue for async processing
    Capsule::table('mod_async_hooks')->insert([
        'hook_name' => 'InvoicePaid',
        'payload' => json_encode($vars),
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
        'priority' => 10,
    ]);
});

// Separate cron job processes the queue
add_hook('HourlyCronJob', 1, function($vars) {
    $pending = Capsule::table('mod_async_hooks')
        ->where('status', 'pending')
        ->where('created_at', '<', date('Y-m-d H:i:s', strtotime('-1 minute')))
        ->orderBy('priority', 'ASC')
        ->limit(50)
        ->get();

    foreach ($pending as $job) {
        // Mark as processing
        Capsule::table('mod_async_hooks')
            ->where('id', $job->id)
            ->update(['status' => 'processing', 'started_at' => date('Y-m-d H:i:s')]);

        try {
            $payload = json_decode($job->payload, true);
            processAsyncHook($job->hook_name, $payload);

            Capsule::table('mod_async_hooks')
                ->where('id', $job->id)
                ->update(['status' => 'completed', 'completed_at' => date('Y-m-d H:i:s')]);

        } catch (\Exception $e) {
            Capsule::table('mod_async_hooks')
                ->where('id', $job->id)
                ->update([
                    'status' => 'failed',
                    'error' => $e->getMessage(),
                    'attempts' => $job->attempts + 1,
                ]);
        }
    }
});
```

### 5. Hook-Based Workflow System

```php
<?php
// includes/hooks/WorkflowEngine.php

namespace MyModule\Hooks;

class WorkflowEngine {
    private array $workflows = [];

    public function registerWorkflow(string $name, array $config): void {
        $this->workflows[$name] = $config;
    }

    public function execute(string $workflowName, array $context): void {
        if (!isset($this->workflows[$workflowName])) {
            throw new \Exception("Workflow not found: {$workflowName}");
        }

        $workflow = $this->workflows[$workflowName];

        foreach ($workflow['steps'] as $step) {
            $this->executeStep($step, $context);

            if ($this->shouldStop($workflowName, $context)) {
                break;
            }
        }
    }

    private function executeStep(array $step, array &$context): void {
        $action = $step['action'];

        switch ($action) {
            case 'condition':
                $this->evaluateCondition($step, $context);
                break;
            case 'webhook':
                $this->callWebhook($step, $context);
                break;
            case 'email':
                $this->sendEmail($step, $context);
                break;
            case 'api':
                $this->callApi($step, $context);
                break;
            case 'database':
                $this->executeDbOperation($step, $context);
                break;
        }
    }

    private function evaluateCondition(array $step, array $context): void {
        $condition = $step['condition'];
        // Evaluate condition and update context
        $context['_condition_' . $step['name']] = $this->checkCondition($condition, $context);
    }

    private function checkCondition(array $condition, array $context): bool {
        $field = $condition['field'];
        $operator = $condition['operator'];
        $value = $condition['value'];

        $actualValue = $context[$field] ?? null;

        return match($operator) {
            '==' => $actualValue == $value,
            '!=' => $actualValue != $value,
            '>' => $actualValue > $value,
            '<' => $actualValue < $value,
            'in' => in_array($actualValue, (array) $value),
            'contains' => str_contains($actualValue, $value),
            default => false,
        };
    }

    private function shouldStop(string $workflowName, array $context): bool {
        $workflow = $this->workflows[$workflowName];
        return isset($workflow['stop_on_condition'])
            && $this->checkCondition($workflow['stop_on_condition'], $context);
    }

    private function callWebhook(array $step, array $context): void {
        $url = $this->interpolate($step['url'], $context);
        $payload = $this->interpolate($step['payload'] ?? [], $context);

        // Send webhook
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
        ]);
        curl_exec($ch);
        curl_close($ch);
    }

    private function interpolate($template, array $context) {
        if (is_array($template)) {
            return array_map(fn($v) => $this->interpolate($v, $context), $template);
        }

        if (is_string($template)) {
            return preg_replace_callback('/\{\{(\w+)\}\}/', function($m) use ($context) {
                return $context[$m[1]] ?? '';
            }, $template);
        }

        return $template;
    }

    private function sendEmail(array $step, array $context): void {
        // Email sending logic
    }

    private function callApi(array $step, array $context): void {
        // API call logic
    }

    private function executeDbOperation(array $step, array $context): void {
        // Database operation logic
    }
}

// Initialize workflow engine
$workflowEngine = new \MyModule\Hooks\WorkflowEngine();

// Register a workflow
$workflowEngine->registerWorkflow('new_client_onboarding', [
    'steps' => [
        [
            'name' => 'check_tier',
            'action' => 'condition',
            'condition' => [
                'field' => 'client_type',
                'operator' => '==',
                'value' => 'premium',
            ],
        ],
        [
            'name' => 'send_welcome',
            'action' => 'email',
            'template' => 'welcome_premium',
        ],
        [
            'name' => 'create_accounts',
            'action' => 'api',
            'endpoint' => '/api/accounts',
        ],
        [
            'name' => 'notify_slack',
            'action' => 'webhook',
            'url' => 'https://hooks.slack.com/...',
            'payload' => ['text' => '{{firstname}} {{lastname}} just signed up!'],
        ],
    ],
    'stop_on_condition' => [
        'field' => 'is_test_account',
        'operator' => '==',
        'value' => true,
    ],
]);

// Hook into WHMCS
add_hook('ClientAdd', 10, function($vars) use ($workflowEngine) {
    $workflowEngine->execute('new_client_onboarding', $vars);
});
```

## Complete Hook Reference

### 1. Client Lifecycle Hooks

```php
<?php
// Client Registration & Management
add_hook('ClientAdd', 1, function($vars) { /* New client created */ });
add_hook('ClientEdit', 1, function($vars) { /* Client details updated */ });
add_hook('ClientDelete', 1, function($vars) { /* Client deleted */ });
add_hook('ClientLogin', 1, function($vars) { /* Client logged in */ });
add_hook('ClientLogout', 1, function($vars) { /* Client logged out */ });
add_hook('ClientChangePassword', 1, function($vars) { /* Password changed */ });
add_hook('ClientTwoFactor', 1, function($vars) { /* 2FA attempted */ });
```

### 2. Order Hooks

```php
<?php
// Order Processing
add_hook('NewOrder', 1, function($vars) { /* New order placed */ });
add_hook('AcceptOrder', 1, function($vars) { /* Order accepted */ });
add_hook('CancelOrder', 1, function($vars) { /* Order cancelled */ });
add_hook('FraudOrder', 1, function($vars) { /* Order flagged as fraud */ });
add_hook('PendingOrder', 1, function($vars) { /* Order pending review */ });
add_hook('OrderPaid', 1, function($vars) { /* Order payment received */ });
```

### 3. Service Hooks

```php
<?php
// Service Lifecycle
add_hook('AfterModuleCreate', 1, function($vars) { /* Service created */ });
add_hook('AfterModuleSuspend', 1, function($vars) { /* Service suspended */ });
add_hook('AfterModuleUnsuspend', 1, function($vars) { /* Service unsuspended */ });
add_hook('AfterModuleTerminate', 1, function($vars) { /* Service terminated */ });
add_hook('AfterModuleChangePassword', 1, function($vars) { /* Password changed */ });
add_hook('AfterModuleChangePackage', 1, function($vars) { /* Package changed */ });
add_hook('ServiceRenewal', 1, function($vars) { /* Service renewed */ });
```

### 4. Invoice Hooks

```php
<?php
// Invoice Processing
add_hook('InvoiceCreation', 1, function($vars) { /* Invoice created */ });
add_hook('InvoicePaid', 1, function($vars) { /* Invoice paid */ });
add_hook('InvoicePendingReview', 1, function($vars) { /* Invoice pending */ });
add_hook('InvoicePaymentFailed', 1, function($vars) { /* Payment failed */ });
add_hook('InvoiceRefunded', 1, function($vars) { /* Invoice refunded */ });
add_hook('InvoiceDeleted', 1, function($vars) { /* Invoice deleted */ });
add_hook('AddTransaction', 1, function($vars) { /* Transaction added */ });
add_hook('DailyCronJob', 1, function($vars) { /* Daily cron */ });
```

### 5. Domain Hooks

```php
<?php
// Domain Operations
add_hook('DomainRegistration', 1, function($vars) { /* Domain registered */ });
add_hook('DomainTransferAccept', 1, function($vars) { /* Transfer accepted */ });
add_hook('DomainTransferReject', 1, function($vars) { /* Transfer rejected */ });
add_hook('DomainRenewal', 1, function($vars) { /* Domain renewed */ });
add_hook('DomainDeletion', 1, function($vars) { /* Domain deleted */ });
add_hook('DomainTransferCompleted', 1, function($vars) { /* Transfer complete */ });
```

### 6. Ticket Hooks

```php
<?php
// Support Tickets
add_hook('TicketOpen', 1, function($vars) { /* New ticket opened */ });
add_hook('TicketReply', 1, function($vars) { /* Ticket replied */ });
add_hook('TicketClose', 1, function($vars) { /* Ticket closed */ });
add_hook('TicketUserReply', 1, function($vars) { /* Client replied */ });
add_hook('TicketAdminReply', 1, function($vars) { /* Admin replied */ });
add_hook('TicketEscalate', 1, function($vars) { /* Ticket escalated */ });
add_hook('TicketFlagged', 1, function($vars) { /* Ticket flagged */ });
```

### 7. Admin Hooks

```php
<?php
// Admin Actions
add_hook('AdminLogin', 1, function($vars) { /* Admin logged in */ });
add_hook('AdminLogout', 1, function($vars) { /* Admin logged out */ });
add_hook('AdminAreaPage', 1, function($vars) { /* Admin page load */ });
add_hook('AdminHomeView', 1, function($vars) { /* Admin dashboard */ });
```

### 8. Email Hooks

```php
<?php
// Email Processing
add_hook('EmailPreSend', 1, function($vars) { /* Before email sent */ });
add_hook('EmailSent', 1, function($vars) { /* Email sent */ });
add_hook('EmailReject', 1, function($vars) { /* Email rejected */ });
```

## Testing Hooks

```php
<?php
// tests/hooks/test_ClientAdd.php

namespace MyModule\Tests;

class ClientAddHookTest {
    private $hook;

    public function setUp(): void {
        // Initialize test hook
        $this->hook = new \MyModule\Hooks\ClientEventHandler();
    }

    public function testClientAddSyncsToExternal(): void {
        $vars = [
            'userid' => 123,
            'firstname' => 'John',
            'lastname' => 'Doe',
            'email' => 'john@example.com',
        ];

        // Mock external API
        $mockApi = $this->createMock(ExternalApi::class);
        $mockApi->expects($this->once())
            ->method('createClient')
            ->with($this->equalTo($vars));

        // Execute hook
        $this->hook->handleClientAdd($vars);

        // Assertions
        $this->assertLogContains("Client added: 123");
    }

    public function testClientAddHandlesApiFailure(): void {
        $vars = [
            'userid' => 123,
            'email' => 'john@example.com',
        ];

        $this->expectException(\Exception::class);
        $this->hook->handleClientAdd($vars);
    }
}
```

## Checklist

- [ ] Hook file created with proper structure
- [ ] WHMCS check included
- [ ] Hook priority set appropriately
- [ ] Error handling with try/catch
- [ ] Logging for debugging
- [ ] Conditional execution where needed
- [ ] Async processing for heavy operations
- [ ] Test coverage for critical hooks

---

**Related Skills:**
- whmcs-hooks-development
- whmcs-cron-automation
- whmcs-logging
- whmcs-notification-builder
- whmcs-addon-builder

**Reference:**
- WHMCS Hook System: https://developers.whmcs.com/hooks/