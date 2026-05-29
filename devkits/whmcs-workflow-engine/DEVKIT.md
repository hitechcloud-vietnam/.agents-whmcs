# WHMCS Workflow Engine Module - DEVKIT

## Module Information
- **Name**: Workflow Engine
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Advanced workflow automation with conditional branching

## Installation
1. Copy to `/modules/addons/workflow_engine/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('OrderPlaced', 1, function($vars) {
    WorkflowEngine::executeWorkflow('new_order', $vars);
});

add_hook('InvoicePaid', 1, function($vars) {
    WorkflowEngine::executeWorkflow('payment_received', $vars);
});

add_hook('ServiceCreated', 1, function($vars) {
    WorkflowEngine::executeWorkflow('service_provisioned', $vars);
});

add_hook('DailyCronJob', 1, function($vars) {
    WorkflowEngine::processPendingTasks();
});
```

### includes/WorkflowEngine.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class WorkflowEngine
{
    private static $workflowsTable = 'mod_workflows';
    private static $executionsTable = 'mod_workflow_executions';
    private static $tasksTable = 'mod_workflow_tasks';
    
    public static function createWorkflow($config)
    {
        $data = [
            'name' => $config['name'] ?? 'Untitled Workflow',
            'trigger' => $config['trigger'] ?? 'manual',
            'definition' => json_encode($config['definition'] ?? []),
            'enabled' => $config['enabled'] ?? true,
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$workflowsTable . "
            (name, trigger, definition, enabled, created_at)
            VALUES (
                '" . db_escape_string($data['name']) . "',
                '" . db_escape_string($data['trigger']) . "',
                '" . db_escape_string($data['definition']) . "',
                " . ($data['enabled'] ? 1 : 0) . ",
                NOW()
            )
        ");
        
        return mysql_insert_id();
    }
    
    public static function executeWorkflow($trigger, $context)
    {
        $workflows = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$workflowsTable . "
            WHERE trigger = '" . db_escape_string($trigger) . "'
            AND enabled = 1
        ");
        
        while ($workflow = mysql_fetch_array($workflows)) {
            self::runWorkflow($workflow, $context);
        }
    }
    
    public static function runWorkflow($workflow, $context)
    {
        $executionId = self::startExecution($workflow['id'], $context);
        
        $definition = json_decode($workflow['definition'], true);
        
        $result = self::executeNodes($definition['nodes'] ?? [], $definition['edges'] ?? [], $context);
        
        self::completeExecution($executionId, $result);
        
        return $result;
    }
    
    private function executeNodes($nodes, $edges, $context)
    {
        $results = [];
        $currentNode = self::findStartNode($nodes);
        
        while ($currentNode) {
            $nodeResult = self::executeNode($currentNode, $context);
            $results[$currentNode['id']] = $nodeResult;
            
            if (!$nodeResult['success']) {
                break;
            }
            
            $nextNodeId = self::findNextNode($currentNode['id'], $edges, $nodeResult);
            $currentNode = $nextNodeId ? self::findNodeById($nodes, $nextNodeId) : null;
        }
        
        return $results;
    }
    
    private function findStartNode($nodes)
    {
        foreach ($nodes as $node) {
            if ($node['type'] === 'start') {
                return $node;
            }
        }
        return null;
    }
    
    private function findNodeById($nodes, $id)
    {
        foreach ($nodes as $node) {
            if ($node['id'] === $id) {
                return $node;
            }
        }
        return null;
    }
    
    private function findNextNode($currentId, $edges, $nodeResult)
    {
        foreach ($edges as $edge) {
            if ($edge['source'] === $currentId) {
                $condition = $edge['condition'] ?? 'default';
                
                if ($condition === 'default' || 
                    ($condition === 'success' && $nodeResult['success']) ||
                    ($condition === 'failure' && !$nodeResult['success'])) {
                    return $edge['target'];
                }
            }
        }
        return null;
    }
    
    private function executeNode($node, $context)
    {
        $nodeType = $node['type'] ?? 'action';
        
        switch ($nodeType) {
            case 'condition':
                return self::evaluateCondition($node, $context);
                
            case 'action':
                return self::performAction($node, $context);
                
            case 'delay':
                return self::scheduleDelay($node, $context);
                
            case 'notification':
                return self::sendNotification($node, $context);
                
            case 'api_call':
                return self::makeApiCall($node, $context);
                
            case 'end':
                return ['success' => true, 'done' => true];
                
            default:
                return ['success' => false, 'error' => 'Unknown node type'];
        }
    }
    
    private function evaluateCondition($node, $context)
    {
        $conditions = $node['conditions'] ?? [];
        
        foreach ($conditions as $condition) {
            $field = $condition['field'];
            $operator = $condition['operator'];
            $value = $condition['value'];
            
            $contextValue = self::getNestedValue($context, $field);
            
            $result = self::compareValues($contextValue, $operator, $value);
            
            if (!$result && $condition['required'] ?? false) {
                return ['success' => false, 'error' => 'Condition not met'];
            }
        }
        
        return ['success' => true, 'next' => $node['next'] ?? null];
    }
    
    private function getNestedValue($array, $path)
    {
        $keys = explode('.', $path);
        $value = $array;
        
        foreach ($keys as $key) {
            if (is_array($value) && isset($value[$key])) {
                $value = $value[$key];
            } else {
                return null;
            }
        }
        
        return $value;
    }
    
    private function compareValues($value, $operator, $expected)
    {
        switch ($operator) {
            case 'equals':
                return $value == $expected;
            case 'not_equals':
                return $value != $expected;
            case 'greater_than':
                return $value > $expected;
            case 'less_than':
                return $value < $expected;
            case 'contains':
                return strpos($value, $expected) !== false;
            case 'starts_with':
                return strpos($value, $expected) === 0;
            case 'ends_with':
                return strpos($value, $expected) === strlen($value) - strlen($expected);
            case 'is_empty':
                return empty($value);
            case 'is_not_empty':
                return !empty($value);
            default:
                return false;
        }
    }
    
    private function performAction($node, $context)
    {
        $actionType = $node['action_type'] ?? 'log';
        
        switch ($actionType) {
            case 'send_email':
                $clientId = $context['userid'] ?? 0;
                $client = full_query("SELECT email FROM " . TABLE_PREFIX . "tblclients WHERE id = " . (int)$clientId);
                $email = mysql_fetch_array($client)['email'] ?? '';
                
                if ($email) {
                    send_email($email, $node['template'] ?? 'generic', [
                        'client_name' => $context['firstname'] ?? 'Customer',
                        'custom_data' => $node['data'] ?? []
                    ]);
                }
                break;
                
            case 'add_credit':
                $clientId = $context['userid'] ?? 0;
                $amount = $node['amount'] ?? 0;
                
                full_query("
                    UPDATE " . TABLE_PREFIX . "tblclients
                    SET credit = credit + " . (float)$amount . "
                    WHERE id = " . (int)$clientId
                );
                break;
                
            case 'update_status':
                $serviceId = $context['serviceid'] ?? 0;
                $status = $node['status'] ?? 'Active';
                
                full_query("
                    UPDATE " . TABLE_PREFIX . "tblhosting
                    SET domainstatus = '" . db_escape_string($status) . "'
                    WHERE id = " . (int)$serviceId
                );
                break;
                
            case 'create_invoice':
                $clientId = $context['userid'] ?? 0;
                $amount = $node['amount'] ?? 0;
                
                createInvoices($clientId);
                break;
                
            case 'log':
            default:
                logActivity("Workflow action: " . ($node['description'] ?? 'No description'));
        }
        
        return ['success' => true];
    }
    
    private function scheduleDelay($node, $context)
    {
        $delay = $node['duration'] ?? 60;
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$tasksTable . "
            (workflow_id, context, scheduled_for, created_at)
            VALUES (
                " . (int)($context['workflow_id'] ?? 0) . ",
                '" . db_escape_string(json_encode($context)) . "',
                DATE_ADD(NOW(), INTERVAL " . (int)$delay . " SECOND),
                NOW()
            )
        ");
        
        return ['success' => true, 'delayed' => true];
    }
    
    private function sendNotification($node, $context)
    {
        $channels = $node['channels'] ?? ['email'];
        
        foreach ($channels as $channel) {
            switch ($channel) {
                case 'email':
                    $clientId = $context['userid'] ?? 0;
                    $client = full_query("SELECT email FROM " . TABLE_PREFIX . "tblclients WHERE id = " . (int)$clientId);
                    $email = mysql_fetch_array($client)['email'] ?? '';
                    if ($email) {
                        send_email($email, $node['template'] ?? 'notification', $context);
                    }
                    break;
                    
                case 'admin':
                    send_admin_notification(
                        $node['admin_role'] ?? 'all',
                        $node['subject'] ?? 'Workflow Notification',
                        $node['message'] ?? '',
                        $context
                    );
                    break;
                    
                case 'webhook':
                    $url = $node['webhook_url'] ?? '';
                    if ($url) {
                        WorkflowHelper::postWebhook($url, $context);
                    }
                    break;
            }
        }
        
        return ['success' => true];
    }
    
    private function makeApiCall($node, $context)
    {
        $url = $node['url'] ?? '';
        $method = strtoupper($node['method'] ?? 'GET');
        $headers = $node['headers'] ?? [];
        $body = $node['body'] ?? [];
        
        return WorkflowHelper::callApi($url, $method, $headers, $body);
    }
    
    private function startExecution($workflowId, $context)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$executionsTable . "
            (workflow_id, context, status, started_at)
            VALUES (
                " . (int)$workflowId . ",
                '" . db_escape_string(json_encode($context)) . "',
                'running',
                NOW()
            )
        ");
        
        return mysql_insert_id();
    }
    
    private function completeExecution($executionId, $results)
    {
        full_query("
            UPDATE " . TABLE_PREFIX . self::$executionsTable . "
            SET status = 'completed',
                completed_at = NOW(),
                results = '" . db_escape_string(json_encode($results)) . "'
            WHERE id = " . (int)$executionId
        );
    }
    
    public static function processPendingTasks()
    {
        $tasks = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$tasksTable . "
            WHERE status = 'pending'
            AND scheduled_for <= NOW()
            LIMIT 100
        ");
        
        while ($task = mysql_fetch_array($tasks)) {
            $workflow = full_query("SELECT * FROM " . TABLE_PREFIX . self::$workflowsTable . " WHERE id = " . (int)$task['workflow_id']);
            $workflowData = mysql_fetch_array($workflow);
            
            $context = json_decode($task['context'], true);
            self::runWorkflow($workflowData, $context);
            
            full_query("
                UPDATE " . TABLE_PREFIX . self::$tasksTable . "
                SET status = 'completed',
                    completed_at = NOW()
                WHERE id = " . (int)$task['id']
            );
        }
    }
}

class WorkflowHelper
{
    public static function callApi($url, $method, $headers, $body)
    {
        $ch = curl_init();
        
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 30);
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
        
        if (!empty($headers)) {
            curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
        }
        
        if (!empty($body) && in_array($method, ['POST', 'PUT', 'PATCH'])) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($body));
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'status_code' => $httpCode,
            'response' => $response
        ];
    }
    
    public static function postWebhook($url, $data)
    {
        $ch = curl_init();
        
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        
        curl_exec($ch);
        curl_close($ch);
    }
}

function workflow_engine_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_workflows (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            trigger VARCHAR(100),
            definition TEXT,
            enabled TINYINT(1) DEFAULT 1,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_workflow_executions (
            id INT AUTO_INCREMENT PRIMARY KEY,
            workflow_id INT NOT NULL,
            context TEXT,
            results TEXT,
            status VARCHAR(20),
            started_at DATETIME,
            completed_at DATETIME
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_workflow_tasks (
            id INT AUTO_INCREMENT PRIMARY KEY,
            workflow_id INT NOT NULL,
            context TEXT,
            status VARCHAR(20) DEFAULT 'pending',
            scheduled_for DATETIME,
            completed_at DATETIME
        )
    ");
    
    return ['status' => 'success', 'description' => 'Workflow Engine activated'];
}

function workflow_engine_deactivate()
{
    return ['status' => 'success', 'description' => 'Workflow Engine deactivated'];
}

function workflow_engine_config()
{
    return [
        'name' => 'Workflow Engine',
        'description' => 'Advanced workflow automation with conditional branching',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'max_execution_time' => [
                'Type' => 'text',
                'FriendlyName' => 'Max Execution Time (seconds)',
                'Default' => '300'
            ],
            'allow_webhooks' => [
                'Type' => 'yesno',
                'FriendlyName' => 'Allow Webhook Actions',
                'Default' => '1'
            ]
        ]
    ];
}
```