# WHMCS Trigger Chains Workflow

## Overview
This workflow implements multi-step trigger chains for complex automated workflows in WHMCS.

## Prerequisites
- WHMCS with custom hooks capability
- PHP 7.4+ for modern syntax
- Database access for chain state storage

## Step-by-Step Process

### Step 1: Understand Trigger Chain Architecture
```
TRIGGER → STEP 1 → [Success] → STEP 2 → [Success] → STEP 3 → COMPLETE
                        ↓
                    [Failure]
                        ↓
                   ERROR HANDLER
                        ↓
                   STEP A (Rollback)
                        ↓
                   NOTIFY
```

### Step 2: Create Chain Database Schema
```php
<?php
// /includes/hooks/chain_schema.php

function installTriggerChainSchema()
{
    // Trigger chains table
    Capsule::schema()->create('mod_trigger_chains', function (Blueprint $table) {
        $table->increments('id');
        $table->string('name', 100);
        $table->string('trigger_event', 100);
        $table->json('steps');
        $table->json('config');
        $table->enum('status', ['active', 'paused', 'disabled'])->default('active');
        $table->timestamps();
    });

    // Chain executions table
    Capsule::schema()->create('mod_chain_executions', function (Blueprint $table) {
        $table->increments('id');
        $table->unsignedInteger('chain_id');
        $table->unsignedInteger('entity_id');
        $table->string('entity_type', 50);
        $table->json('context');
        $table->enum('status', ['running', 'completed', 'failed', 'cancelled']);
        $table->unsignedInteger('current_step');
        $table->json('step_results');
        $table->json('errors');
        $table->timestamp('started_at');
        $table->timestamp('completed_at')->nullable();
        $table->timestamps();

        $table->foreign('chain_id')->references('id')->on('mod_trigger_chains');
        $table->index(['entity_type', 'entity_id']);
    });

    // Chain execution log
    Capsule::schema()->create('mod_chain_logs', function (Blueprint $table) {
        $table->increments('id');
        $table->unsignedInteger('execution_id');
        $table->unsignedInteger('step_index');
        $table->string('step_name', 100);
        $table->enum('status', ['pending', 'running', 'completed', 'failed', 'skipped']);
        $table->text('input');
        $table->text('output');
        $table->text('error')->nullable();
        $table->integer('duration_ms');
        $table->timestamp('executed_at');

        $table->foreign('execution_id')->references('id')->on('mod_chain_executions');
    });
}
```

### Step 3: Create Trigger Chain Manager
```php
<?php
// /includes/triggers/TriggerChainManager.php

namespace WHMCS\Triggers;

class TriggerChainManager
{
    /**
     * Execute a trigger chain
     */
    public function execute(string $chainName, int $entityId, string $entityType, array $context = [])
    {
        $chain = $this->getChain($chainName);

        if (!$chain || $chain['status'] !== 'active') {
            return ['success' => false, 'error' => 'Chain not found or inactive'];
        }

        // Create execution record
        $executionId = $this->createExecution($chain['id'], $entityId, $entityType, $context);

        // Execute chain steps
        return $this->runChain($executionId, $chain['steps'], $context);
    }

    /**
     * Run all steps in the chain
     */
    private function runChain(int $executionId, array $steps, array $context): array
    {
        $results = [];
        $errors = [];
        $currentStep = 0;

        // Update execution status
        $this->updateExecution($executionId, ['status' => 'running', 'current_step' => 0]);

        foreach ($steps as $index => $step) {
            $currentStep = $index;
            $stepStart = microtime(true);

            $this->logStep($executionId, $index, $step['name'], 'running');

            try {
                // Execute step
                $result = $this->executeStep($step, $context);

                $duration = round((microtime(true) - $stepStart) * 1000);

                $results[$step['name']] = [
                    'success' => true,
                    'data' => $result,
                    'duration_ms' => $duration
                ];

                // Update context with step results for next steps
                $context[$step['name']] = $result;

                $this->logStep($executionId, $index, $step['name'], 'completed', $result, null, $duration);

                // Check step conditions for branching
                if (isset($step['on_failure']) && !$result['success']) {
                    $fallbackSteps = $this->getStepsByName($steps, $step['on_failure']);
                    foreach ($fallbackSteps as $fallbackIndex => $fallbackStep) {
                        $results = array_merge(
                            $results,
                            $this->runChainSegment($executionId, $steps, $fallbackIndex, $context)
                        );
                    }
                }
            } catch (Exception $e) {
                $duration = round((microtime(true) - $stepStart) * 1000);

                $errors[] = [
                    'step' => $step['name'],
                    'error' => $e->getMessage()
                ];

                $this->logStep($executionId, $index, $step['name'], 'failed', null, $e->getMessage(), $duration);

                // Execute error handler if defined
                if (isset($step['error_handler'])) {
                    $handlerSteps = $this->getStepsByName($steps, $step['error_handler']);
                    foreach ($handlerSteps as $handlerStep) {
                        $this->executeStep($handlerStep, array_merge($context, ['error' => $e]));
                    }
                }

                // Check if chain should stop on error
                if (!($step['continue_on_error'] ?? false)) {
                    break;
                }
            }
        }

        // Update final execution status
        $finalStatus = empty($errors) ? 'completed' : 'failed';
        $this->updateExecution($executionId, [
            'status' => $finalStatus,
            'current_step' => $currentStep,
            'step_results' => json_encode($results),
            'errors' => json_encode($errors),
            'completed_at' => date('Y-m-d H:i:s')
        ]);

        return [
            'success' => empty($errors),
            'execution_id' => $executionId,
            'results' => $results,
            'errors' => $errors
        ];
    }

    /**
     * Execute a single step
     */
    private function executeStep(array $step, array $context)
    {
        $action = $step['action'];
        $params = $step['params'] ?? [];

        // Substitute context variables in params
        $params = $this->substituteVariables($params, $context);

        return match ($action) {
            // Email actions
            'send_email' => $this->actionSendEmail($params),
            'send_sms' => $this->actionSendSMS($params),

            // Service actions
            'activate_service' => $this->actionActivateService($params),
            'suspend_service' => $this->actionSuspendService($params),
            'terminate_service' => $this->actionTerminateService($params),

            // Invoice actions
            'create_invoice' => $this->actionCreateInvoice($params),
            'send_invoice' => $this->actionSendInvoice($params),
            'apply_credit' => $this->actionApplyCredit($params),

            // External actions
            'http_request' => $this->actionHttpRequest($params),
            'webhook' => $this->actionWebhook($params),
            'sync_crm' => $this->actionSyncCRM($params),

            // Logic actions
            'condition' => $this->actionCondition($params),
            'delay' => $this->actionDelay($params),
            'log' => $this->actionLog($params),

            default => throw new Exception("Unknown action: {$action}")
        };
    }

    private function actionSendEmail(array $params)
    {
        sendEmail($params['client_id'], $params['template'], $params['vars'] ?? []);

        return ['success' => true, 'message' => 'Email sent'];
    }

    private function actionSuspendService(array $params)
    {
        $result = localApi('SuspendAccount', [
            'serviceid' => $params['service_id'],
            'suspendreason' => $params['reason'] ?? 'Automated suspension'
        ]);

        return ['success' => true, 'result' => $result];
    }

    private function actionWebhook(array $params)
    {
        $ch = curl_init($params['url']);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($params['payload'] ?? []),
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'http_code' => $httpCode,
            'response' => $response
        ];
    }

    private function actionDelay(array $params)
    {
        sleep($params['seconds'] ?? 60);
        return ['success' => true, 'waited_seconds' => $params['seconds']];
    }

    private function substituteVariables(array $params, array $context): array
    {
        array_walk_recursive($params, function (&$value) use ($context) {
            if (is_string($value) && preg_match('/\{([^}]+)\}/', $value, $matches)) {
                $key = $matches[1];
                $value = $context[$key] ?? $value;
            }
        });

        return $params;
    }

    private function getStepsByName(array $steps, string $name): array
    {
        return array_filter($steps, fn($step) => $step['name'] === $name);
    }
}
```

### Step 4: Create Common Chain Templates

#### New Client Onboarding Chain
```php
<?php
// Chain: new_client_onboarding

$onboardingChain = [
    'name' => 'New Client Onboarding',
    'trigger_event' => 'ClientAdd',
    'status' => 'active',
    'steps' => [
        [
            'name' => 'send_welcome_email',
            'action' => 'send_email',
            'params' => [
                'template' => 'Welcome Email',
                'client_id' => '{client_id}',
                'vars' => [
                    'first_name' => '{first_name}',
                    'email' => '{email}'
                ]
            ]
        ],
        [
            'name' => 'create_welcome_ticket',
            'action' => 'http_request',
            'params' => [
                'url' => 'https://api.internal.com/tickets',
                'payload' => [
                    'subject' => 'Welcome to our service!',
                    'client_id' => '{client_id}'
                ]
            ]
        ],
        [
            'name' => 'delay_for_followup',
            'action' => 'delay',
            'params' => ['seconds' => 86400] // 24 hours
        ],
        [
            'name' => 'send_followup_email',
            'action' => 'send_email',
            'params' => [
                'template' => 'Onboarding Follow-up',
                'client_id' => '{client_id}'
            ],
            'continue_on_error' => true
        ],
        [
            'name' => 'add_to_crm',
            'action' => 'sync_crm',
            'params' => [
                'system' => 'hubspot',
                'action' => 'create_contact',
                'data' => '{client_data}'
            ]
        ]
    ],
    'config' => [
        'timeout' => 3600,
        'retry_on_failure' => true
    ]
];
```

#### Service Activation Chain
```php
<?php
// Chain: service_activation

$activationChain = [
    'name' => 'Service Activation',
    'trigger_event' => 'AfterServiceCreate',
    'status' => 'active',
    'steps' => [
        [
            'name' => 'validate_order',
            'action' => 'condition',
            'params' => [
                'check' => 'fraud_score',
                'threshold' => 80,
                'pass_action' => 'provision_service',
                'fail_action' => 'flag_for_review'
            ]
        ],
        [
            'name' => 'provision_service',
            'action' => 'http_request',
            'params' => [
                'url' => 'https://api.provisioning.com/create',
                'payload' => ['service_id' => '{service_id}']
            ]
        ],
        [
            'name' => 'activate_whmcs_service',
            'action' => 'activate_service',
            'params' => ['service_id' => '{service_id}']
        ],
        [
            'name' => 'send_activation_email',
            'action' => 'send_email',
            'params' => [
                'template' => 'Service Activated',
                'client_id' => '{client_id}'
            ]
        ],
        [
            'name' => 'sync_to_crm',
            'action' => 'sync_crm',
            'params' => [
                'system' => 'salesforce',
                'action' => 'update_opportunity',
                'service_id' => '{service_id}'
            ]
        ],
        [
            'name' => 'error_handler',
            'action' => 'log',
            'params' => [
                'message' => 'Service activation failed for {service_id}',
                'level' => 'error'
            ],
            'error_handler' => 'handle_activation_failure'
        ]
    ]
];
```

### Step 5: Create Chain Execution Hooks
```php
<?php
// /includes/hooks/chain_hooks.php

use WHMCS\Triggers\TriggerChainManager;

$chainManager = new TriggerChainManager();

// Hook: Client created
add_hook('ClientAdd', 1, function($vars) use ($chainManager) {
    $client = getClientsDetails($vars['userid']);

    $chainManager->execute('new_client_onboarding', $vars['userid'], 'client', [
        'client_id' => $vars['userid'],
        'first_name' => $client['firstname'],
        'email' => $client['email'],
        'client_data' => $client
    ]);
});

// Hook: Service created
add_hook('AfterServiceCreate', 1, function($vars) use ($chainManager) {
    $chainManager->execute('service_activation', $vars['serviceid'], 'service', [
        'service_id' => $vars['serviceid'],
        'client_id' => $vars['userid'],
        'product_id' => $vars['packageid']
    ]);
});

// Hook: Invoice paid
add_hook('InvoicePaid', 1, function($vars) use ($chainManager) {
    $chainManager->execute('invoice_paid_actions', $vars['invoiceid'], 'invoice', [
        'invoice_id' => $vars['invoiceid'],
        'client_id' => $vars['userid']
    ]);
});
```

### Step 6: Create Chain Definition API
```php
<?php
// Admin API endpoints for chain management

/**
 * GET /api/chains - List all trigger chains
 */
function apiListChains()
{
    $chains = Capsule::table('mod_trigger_chains')
        ->orderBy('name')
        ->get();

    return ['chains' => $chains];
}

/**
 * POST /api/chains - Create new chain
 */
function apiCreateChain(Request $request)
{
    $data = $request->validate([
        'name' => 'required|string',
        'trigger_event' => 'required|string',
        'steps' => 'required|array',
        'config' => 'array'
    ]);

    $chainId = Capsule::table('mod_trigger_chains')->insertGetId([
        'name' => $data['name'],
        'trigger_event' => $data['trigger_event'],
        'steps' => json_encode($data['steps']),
        'config' => json_encode($data['config'] ?? []),
        'status' => 'active',
        'created_at' => date('Y-m-d H:i:s')
    ]);

    return ['chain_id' => $chainId, 'success' => true];
}

/**
 * POST /api/chains/{id}/execute - Manually execute chain
 */
function apiExecuteChain(Request $request, int $chainId)
{
    $chain = Capsule::table('mod_trigger_chains')
        ->where('id', $chainId)
        ->first();

    if (!$chain) {
        return ['success' => false, 'error' => 'Chain not found'];
    }

    $chainManager = new TriggerChainManager();
    $result = $chainManager->execute(
        $chain->name,
        $request->input('entity_id'),
        $request->input('entity_type', 'unknown'),
        $request->input('context', [])
    );

    return $result;
}
```

### Step 7: Monitor Chain Execution
```php
/**
 * Get chain execution status
 */
function getChainExecutionStatus(int $executionId)
{
    return Capsule::table('mod_chain_executions')
        ->where('id', $executionId)
        ->first();
}

/**
 * Get chain execution logs
 */
function getChainExecutionLogs(int $executionId)
{
    return Capsule::table('mod_chain_logs')
        ->where('execution_id', $executionId)
        ->orderBy('executed_at')
        ->get();
}

/**
 * Admin dashboard widget
 */
add_hook('AdminHomepage', 1, function($vars) {
    $runningExecutions = Capsule::table('mod_chain_executions')
        ->where('status', 'running')
        ->count();

    $failedToday = Capsule::table('mod_chain_executions')
        ->where('status', 'failed')
        ->whereDate('updated_at', date('Y-m-d'))
        ->count();

    return [
        'chainStatus' => [
            'running' => $runningExecutions,
            'failed_today' => $failedToday
        ]
    ];
});
```

## Chain Configuration Example
```json
{
  "name": "Payment Recovery",
  "trigger_event": "InvoiceOverdue",
  "steps": [
    {
      "name": "check_overdue_days",
      "action": "condition",
      "params": {
        "field": "{days_overdue}",
        "operator": ">=",
        "value": 7
      }
    },
    {
      "name": "send_reminder",
      "action": "send_email",
      "params": {
        "template": "Payment Reminder",
        "client_id": "{client_id}"
      }
    },
    {
      "name": "suspend_if_14_days",
      "action": "condition",
      "params": {
        "field": "{days_overdue}",
        "operator": ">=",
        "value": 14
      }
    },
    {
      "name": "suspend_service",
      "action": "suspend_service",
      "params": {
        "service_id": "{service_id}"
      },
      "condition" => "suspend_if_14_days"
    }
  ]
}
```

## Best Practices

1. **Idempotency** - Make steps safe to re-run
2. **Timeout** - Set reasonable timeouts per step
3. **Error Handling** - Always define error handlers
4. **Logging** - Log all step inputs/outputs
5. **Branching** - Use conditions for workflow paths
6. **Monitoring** - Track execution times

## Related Workflows
- [WHMCS Event-Driven Automation](./whmcs-event-driven-automation.md)
- [WHMCS Automation Rules](./whmcs-automation-rules.md)
- [WHMCS Condition Actions](./whmcs-condition-actions.md)