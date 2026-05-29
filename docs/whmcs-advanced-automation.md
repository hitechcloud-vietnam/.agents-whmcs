# WHMCS Advanced Automation

Complete guide to workflow automation.

## Overview

Build sophisticated automated workflows for WHMCS.

## Workflow Engine

### Workflow Manager

```php
<?php
/**
 * Workflow engine
 */
class WorkflowEngine
{
    private array $workflows = [];
    private array $triggers = [];
    private array $actions = [];
    
    /**
     * Register workflow
     */
    public function registerWorkflow(array $workflow): void
    {
        $this->workflows[$workflow['id']] = $workflow;
    }
    
    /**
     * Register trigger
     */
    public function registerTrigger(string $name, callable $handler): void
    {
        $this->triggers[$name] = $handler;
    }
    
    /**
     * Register action
     */
    public function registerAction(string $name, callable $handler): void
    {
        $this->actions[$name] = $handler;
    }
    
    /**
     * Execute workflow
     */
    public function execute(string $workflowId, array $context): array
    {
        if (!isset($this->workflows[$workflowId])) {
            throw new Exception("Workflow not found: {$workflowId}");
        }
        
        $workflow = $this->workflows[$workflowId];
        $results = [];
        
        foreach ($workflow['steps'] as $step) {
            $result = $this->executeStep($step, $context);
            $results[$step['id']] = $result;
            
            // Check condition for next step
            if (isset($step['condition'])) {
                if (!$this->evaluateCondition($step['condition'], $result, $context)) {
                    continue;
                }
            }
            
            // Check stop condition
            if (isset($step['stopIf']) && $this->evaluateCondition($step['stopIf'], $result, $context)) {
                break;
            }
        }
        
        return $results;
    }
    
    /**
     * Execute step
     */
    private function executeStep(array $step, array &$context): mixed
    {
        $actionName = $step['action'];
        
        if (!isset($this->actions[$actionName])) {
            throw new Exception("Action not found: {$actionName}");
        }
        
        $handler = $this->actions[$actionName];
        $params = $this->resolveParams($step['params'] ?? [], $context);
        
        return $handler($params, $context);
    }
    
    /**
     * Resolve parameters with context
     */
    private function resolveParams(array $params, array $context): array
    {
        $resolved = [];
        
        foreach ($params as $key => $value) {
            if (is_string($value) && strpos($value, '{{') !== false) {
                // Replace template variables
                $resolved[$key] = preg_replace_callback(
                    '/\{\{(\w+)\}\}/',
                    fn($matches) => $context[$matches[1]] ?? '',
                    $value
                );
            } else {
                $resolved[$key] = $value;
            }
        }
        
        return $resolved;
    }
    
    /**
     * Evaluate condition
     */
    private function evaluateCondition(array $condition, mixed $result, array $context): bool
    {
        $left = $this->resolveValue($condition['left'] ?? '', $result, $context);
        $right = $this->resolveValue($condition['right'] ?? '', $result, $context);
        $operator = $condition['operator'] ?? '==';
        
        return match($operator) {
            '==' => $left == $right,
            '===' => $left === $right,
            '!=' => $left != $right,
            '>' => $left > $right,
            '<' => $left < $right,
            '>=' => $left >= $right,
            '<=' => $left <= $right,
            'contains' => strpos($left, $right) !== false,
            'starts_with' => strpos($left, $right) === 0,
            'in' => in_array($left, (array)$right),
            'empty' => empty($left),
            'not_empty' => !empty($left),
            default => false,
        };
    }
    
    /**
     * Resolve value from context
     */
    private function resolveValue(string $value, mixed $result, array $context): mixed
    {
        if (strpos($value, 'result.') === 0) {
            $key = substr($value, 7);
            return $result[$key] ?? null;
        }
        
        if (strpos($value, 'context.') === 0) {
            $key = substr($value, 8);
            return $context[$key] ?? null;
        }
        
        return $value;
    }
}
```

## Built-in Actions

### Action Registry

```php
<?php
/**
 * Register built-in actions
 */
$workflow = new WorkflowEngine();

// Email action
$workflow->registerAction('send_email', function($params, &$context) {
    $to = $params['to'];
    $subject = $params['subject'];
    $body = $params['body'];
    
    sendEmailTemplate($to, $subject, $body, $params['template'] ?? 'default');
    
    return ['sent' => true, 'to' => $to];
});

// API action
$workflow->registerAction('call_api', function($params, &$context) {
    $ch = curl_init($params['url']);
    curl_setopt_array($ch, [
        CURLOPT_POST => !empty($params['data']),
        CURLOPT_POSTFIELDS => json_encode($params['data'] ?? []),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    return ['success' => true, 'response' => json_decode($response, true)];
});

// Database action
$workflow->registerAction('db_query', function($params, &$context) {
    $result = Capsule::table($params['table'])
        ->where($params['where_field'] ?? 'id', $params['where_value'] ?? '')
        ->update($params['data'] ?? []);
    
    return ['affected' => $result];
});

// Wait action
$workflow->registerAction('wait', function($params, &$context) {
    sleep($params['seconds'] ?? 60);
    return ['waited' => $params['seconds'] ?? 60];
});

// Condition action
$workflow->registerAction('condition', function($params, &$context) {
    $conditionMet = $this->evaluateCondition($params['condition'], [], $context);
    return ['condition_met' => $conditionMet];
});
```

## Workflow Examples

### New Client Welcome Workflow

```php
<?php
/**
 * Welcome workflow
 */
$workflow->registerWorkflow([
    'id' => 'welcome_client',
    'name' => 'New Client Welcome',
    'trigger' => 'client.created',
    'steps' => [
        [
            'id' => 'send_welcome_email',
            'action' => 'send_email',
            'params' => [
                'to' => '{{email}}',
                'subject' => 'Welcome to our service!',
                'template' => 'welcome',
            ],
        ],
        [
            'id' => 'wait_1_day',
            'action' => 'wait',
            'params' => [
                'seconds' => 86400,
            ],
        ],
        [
            'id' => 'send_intro_email',
            'action' => 'send_email',
            'params' => [
                'to' => '{{email}}',
                'subject' => 'Getting started guide',
                'template' => 'getting_started',
            ],
        ],
        [
            'id' => 'create_welcome_ticket',
            'action' => 'create_ticket',
            'params' => [
                'client_id' => '{{client_id}}',
                'subject' => 'Welcome - How can we help?',
            ],
        ],
    ],
]);
```

### Invoice Follow-up Workflow

```php
<?php
/**
 * Invoice follow-up workflow
 */
$workflow->registerWorkflow([
    'id' => 'invoice_followup',
    'name' => 'Invoice Follow-up',
    'trigger' => 'invoice.overdue',
    'steps' => [
        [
            'id' => 'check_days_overdue',
            'action' => 'condition',
            'params' => [
                'condition' => [
                    'left' => '{{days_overdue}}',
                    'operator' => '>=',
                    'right' => '1',
                ],
            ],
        ],
        [
            'id' => 'send_reminder_1',
            'action' => 'send_email',
            'condition' => ['days_overdue', '==', '1'],
            'params' => [
                'to' => '{{client_email}}',
                'subject' => 'Invoice reminder - Invoice #{{invoice_id}}',
                'template' => 'invoice_reminder_1',
            ],
        ],
        [
            'id' => 'wait_7_days',
            'action' => 'wait',
            'params' => [
                'seconds' => 604800,
            ],
        ],
        [
            'id' => 'send_reminder_2',
            'action' => 'send_email',
            'condition' => ['days_overdue', '>=', '7'],
            'params' => [
                'to' => '{{client_email}}',
                'subject' => 'Final notice - Invoice #{{invoice_id}}',
                'template' => 'invoice_reminder_2',
            ],
        ],
        [
            'id' => 'suspend_service',
            'action' => 'suspend_service',
            'condition' => ['days_overdue', '>=', '14'],
            'params' => [
                'service_id' => '{{service_id}}',
                'reason' => 'Payment overdue',
            ],
        ],
    ],
]);
```

## Trigger System

### Trigger Manager

```php
<?php
/**
 * Trigger manager
 */
class TriggerManager
{
    private WorkflowEngine $workflow;
    
    public function __construct(WorkflowEngine $workflow)
    {
        $this->workflow = $workflow;
    }
    
    /**
     * Register all hooks
     */
    public function registerHooks(): void
    {
        // Client hooks
        add_hook('ClientAdd', 1, function($vars) {
            $this->trigger('client.created', [
                'client_id' => $vars['userid'],
                'email' => Capsule::table('tblclients')->where('id', $vars['userid'])->first()->email,
            ]);
        });
        
        // Invoice hooks
        add_hook('InvoicePaid', 1, function($vars) {
            $this->trigger('invoice.paid', [
                'invoice_id' => $vars['invoice_id'],
            ]);
        });
        
        add_hook('InvoiceOverdue', 1, function($vars) {
            $this->trigger('invoice.overdue', [
                'invoice_id' => $vars['invoice_id'],
                'days_overdue' => $vars['days_overdue'],
            ]);
        });
        
        // Service hooks
        add_hook('AfterModuleCreate', 1, function($vars) {
            $this->trigger('service.created', [
                'service_id' => $vars['serviceid'],
            ]);
        });
    }
    
    /**
     * Trigger workflow
     */
    public function trigger(string $event, array $context): void
    {
        // Find workflows for this trigger
        $workflows = $this->getWorkflowsForTrigger($event);
        
        foreach ($workflows as $workflowId) {
            try {
                $this->workflow->execute($workflowId, $context);
            } catch (Exception $e) {
                logActivity("Workflow {$workflowId} failed: " . $e->getMessage());
            }
        }
    }
    
    /**
     * Get workflows for trigger
     */
    private function getWorkflowsForTrigger(string $trigger): array
    {
        return Capsule::table('mod_workflows')
            ->where('trigger', $trigger)
            ->where('enabled', 1)
            ->pluck('workflow_id')
            ->toArray();
    }
}
```

## Scheduling

### Scheduled Workflows

```php
<?php
/**
 * Schedule workflow execution
 */
function scheduleWorkflow(string $workflowId, array $context, int $executeAt): int
{
    return Capsule::table('mod_workflow_schedules')->insertGetId([
        'workflow_id' => $workflowId,
        'context' => json_encode($context),
        'execute_at' => date('Y-m-d H:i:s', $executeAt),
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Process scheduled workflows
 */
function processScheduledWorkflows(): void
{
    $scheduled = Capsule::table('mod_workflow_schedules')
        ->where('status', 'pending')
        ->where('execute_at', '<=', date('Y-m-d H:i:s'))
        ->get();
    
    $workflow = new WorkflowEngine();
    
    foreach ($scheduled as $schedule) {
        try {
            $context = json_decode($schedule->context, true);
            $workflow->execute($schedule->workflow_id, $context);
            
            Capsule::table('mod_workflow_schedules')
                ->where('id', $schedule->id)
                ->update([
                    'status' => 'completed',
                    'completed_at' => date('Y-m-d H:i:s'),
                ]);
                
        } catch (Exception $e) {
            Capsule::table('mod_workflow_schedules')
                ->where('id', $schedule->id)
                ->update([
                    'status' => 'failed',
                    'error' => $e->getMessage(),
                ]);
        }
    }
}
```

## Best Practices

1. **Idempotent steps** - Handle repeated executions
2. **Error handling** - Graceful failure recovery
3. **Logging** - Track workflow execution
4. **Timeouts** - Set execution timeouts
5. **Retry logic** - Handle transient failures
6. **Monitoring** - Track workflow performance

## Related Documentation

- [whmcs-integration-automation.md](whmcs-integration-automation.md)
- [whmcs-advanced-hooks.md](whmcs-advanced-hooks.md)
