# WHMCS Automation Studio Module - DEVKIT

## Module Information
- **Name**: Automation Studio
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Visual workflow automation builder for WHMCS tasks

## Installation
1. Copy to `/modules/addons/automation_studio/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('DailyCronJob', 1, function($vars) {
    AutomationStudio::executeScheduledTasks();
});

add_hook('InvoiceCreated', 1, function($vars) {
    AutomationStudio::triggerWorkflow('invoice_created', $vars);
});

add_hook('ServiceCreated', 1, function($vars) {
    AutomationStudio::triggerWorkflow('service_created', $vars);
});
```

### includes/AutomationStudio.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class AutomationStudio
{
    private static $table = 'mod_automation_workflows';
    private static $logTable = 'mod_automation_logs';
    
    public static function createWorkflow($name, $trigger, $actions, $conditions = [])
    {
        $workflowData = [
            'name' => $name,
            'trigger_type' => $trigger,
            'actions' => json_encode($actions),
            'conditions' => json_encode($conditions),
            'enabled' => 1,
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$table . "
            (name, trigger_type, actions, conditions, enabled, created_at)
            VALUES (
                '" . db_escape_string($workflowData['name']) . "',
                '" . db_escape_string($workflowData['trigger_type']) . "',
                '" . db_escape_string($workflowData['actions']) . "',
                '" . db_escape_string($workflowData['conditions']) . "',
                1,
                NOW()
            )
        ");
        
        return mysql_insert_id();
    }
    
    public static function triggerWorkflow($triggerType, $context)
    {
        $workflows = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$table . "
            WHERE trigger_type = '" . db_escape_string($triggerType) . "'
            AND enabled = 1
        ");
        
        while ($workflow = mysql_fetch_array($workflows)) {
            self::executeWorkflow($workflow, $context);
        }
    }
    
    public static function executeWorkflow($workflow, $context)
    {
        $conditions = json_decode($workflow['conditions'], true);
        
        if (!empty($conditions) && !self::evaluateConditions($conditions, $context)) {
            return false;
        }
        
        $actions = json_decode($workflow['actions'], true);
        
        foreach ($actions as $action) {
            self::executeAction($action, $context);
        }
        
        self::logExecution($workflow['id'], $context, 'success');
        
        return true;
    }
    
    private static function evaluateConditions($conditions, $context)
    {
        foreach ($conditions as $condition) {
            $field = $condition['field'];
            $operator = $condition['operator'];
            $value = $condition['value'];
            
            $contextValue = $context[$field] ?? null;
            
            switch ($operator) {
                case 'equals':
                    if ($contextValue != $value) return false;
                    break;
                case 'not_equals':
                    if ($contextValue == $value) return false;
                    break;
                case 'contains':
                    if (strpos($contextValue, $value) === false) return false;
                    break;
                case 'greater_than':
                    if ($contextValue <= $value) return false;
                    break;
                case 'less_than':
                    if ($contextValue >= $value) return false;
                    break;
            }
        }
        
        return true;
    }
    
    private static function executeAction($action, $context)
    {
        $actionType = $action['type'];
        
        switch ($actionType) {
            case 'send_email':
                $clientId = $context['userid'] ?? 0;
                $client = get_query_val("tblclients", "email", "id = " . (int)$clientId);
                if ($client) {
                    send_email($client, $action['template'], $action['variables'] ?? []);
                }
                break;
                
            case 'add_credit':
                $clientId = $context['userid'] ?? 0;
                full_query("
                    UPDATE " . TABLE_PREFIX . "tblclients
                    SET credit = credit + " . (float)($action['amount'] ?? 0) . "
                    WHERE id = " . (int)$clientId
                );
                break;
                
            case 'change_status':
                $serviceId = $context['serviceid'] ?? 0;
                full_query("
                    UPDATE " . TABLE_PREFIX . "tblhosting
                    SET domainstatus = '" . db_escape_string($action['status']) . "'
                    WHERE id = " . (int)$serviceId
                );
                break;
                
            case 'create_ticket':
                $clientId = $context['userid'] ?? 0;
                open_ticket([
                    'userid' => $clientId,
                    'deptid' => $action['department'] ?? 1,
                    'subject' => $action['subject'] ?? 'Automated Ticket',
                    'message' => $action['message'] ?? '',
                    'priority' => $action['priority'] ?? 'Medium'
                ]);
                break;
                
            case 'execute_api':
                // Custom API call logic
                $url = $action['url'] ?? '';
                $method = $action['method'] ?? 'GET';
                $data = $action['data'] ?? [];
                
                if ($url) {
                    AutomationHelper::makeApiCall($url, $method, $data);
                }
                break;
                
            case 'run_command':
                $command = $action['command'] ?? '';
                if ($command) {
                    shell_exec($command);
                }
                break;
        }
    }
    
    public static function executeScheduledTasks()
    {
        $scheduled = full_query("
            SELECT * FROM " . TABLE_PREFIX . "mod_automation_scheduled
            WHERE next_run <= NOW()
            AND enabled = 1
        ");
        
        while ($task = mysql_fetch_array($scheduled)) {
            $context = ['scheduled_task' => true, 'task_id' => $task['id']];
            $workflow = [
                'id' => $task['workflow_id'],
                'actions' => $task['actions']
            ];
            
            self::executeWorkflow($workflow, $context);
            
            $nextRun = date('Y-m-d H:i:s', strtotime('+' . $task['interval'], time()));
            full_query("
                UPDATE " . TABLE_PREFIX . "mod_automation_scheduled
                SET next_run = '" . db_escape_string($nextRun) . "'
                WHERE id = " . (int)$task['id']
            );
        }
    }
    
    private static function logExecution($workflowId, $context, $status)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$logTable . "
            (workflow_id, context, status, executed_at)
            VALUES (
                " . (int)$workflowId . ",
                '" . db_escape_string(json_encode($context)) . "',
                '" . db_escape_string($status) . "',
                NOW()
            )
        ");
    }
    
    public static function getWorkflowLogs($workflowId, $limit = 100)
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$logTable . "
            WHERE workflow_id = " . (int)$workflowId . "
            ORDER BY executed_at DESC
            LIMIT " . (int)$limit
        ");
        
        $logs = [];
        while ($row = mysql_fetch_array($result)) {
            $logs[] = $row;
        }
        
        return $logs;
    }
}

class AutomationHelper
{
    public static function makeApiCall($url, $method = 'GET', $data = [])
    {
        $ch = curl_init();
        
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 30);
        
        if ($method == 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($data));
        }
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return $response;
    }
}

function automation_studio_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_automation_workflows (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            trigger_type VARCHAR(100),
            actions TEXT,
            conditions TEXT,
            enabled TINYINT(1) DEFAULT 1,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_automation_logs (
            id INT AUTO_INCREMENT PRIMARY KEY,
            workflow_id INT NOT NULL,
            context TEXT,
            status VARCHAR(20),
            executed_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_automation_scheduled (
            id INT AUTO_INCREMENT PRIMARY KEY,
            workflow_id INT NOT NULL,
            actions TEXT,
            interval VARCHAR(50) DEFAULT '1 day',
            next_run DATETIME,
            enabled TINYINT(1) DEFAULT 1
        )
    ");
    
    return ['status' => 'success', 'description' => 'Automation Studio activated'];
}

function automation_studio_deactivate()
{
    return ['status' => 'success', 'description' => 'Automation Studio deactivated'];
}

function automation_studio_config()
{
    return [
        'name' => 'Automation Studio',
        'description' => 'Visual workflow automation builder for WHMCS',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'max_executions' => [
                'Type' => 'text',
                'FriendlyName' => 'Max Executions per Run',
                'Default' => '100'
            ],
            'execution_timeout' => [
                'Type' => 'text',
                'FriendlyName' => 'Timeout (seconds)',
                'Default' => '60'
            ]
        ]
    ];
}
```