# WHMCS Workflow Automation Skill

## Purpose
Provides patterns for implementing workflow automation in WHMCS, defining automated processes, managing triggers, and handling conditional actions.

## Implementation Patterns

### Workflow Automation
```php
<?php
class WorkflowAutomation {
    private $db;
    
    public function triggerWorkflow($triggerType, $context) {
        $workflows = $this->db->select(
            "SELECT * FROM mod_workflows WHERE trigger_type = ? AND is_active = 1",
            [$triggerType]
        );
        
        foreach ($workflows as $workflow) {
            $this->executeWorkflow($workflow, $context);
        }
    }
    
    private function executeWorkflow($workflow, $context) {
        $conditions = json_decode($workflow['conditions'], true);
        
        if (!$this->evaluateConditions($conditions, $context)) {
            return false;
        }
        
        $actions = json_decode($workflow['actions'], true);
        
        foreach ($actions as $action) {
            $this->executeAction($action, $context);
        }
        
        $this->logExecution($workflow['id'], $context);
        
        return true;
    }
    
    private function evaluateConditions($conditions, $context) {
        foreach ($conditions as $condition) {
            $field = $condition['field'];
            $operator = $condition['operator'];
            $value = $condition['value'];
            
            $actualValue = $this->getNestedValue($context, $field);
            
            if (!$this->compare($actualValue, $operator, $value)) {
                return false;
            }
        }
        return true;
    }
    
    private function executeAction($action, $context) {
        switch ($action['type']) {
            case 'send_email':
                $this->sendEmail($action, $context);
                break;
            case 'update_status':
                $this->updateStatus($action, $context);
                break;
            case 'create_task':
                $this->createTask($action, $context);
                break;
            case 'webhook':
                $this->callWebhook($action, $context);
                break;
        }
    }
    
    private function compare($actual, $operator, $expected) {
        switch ($operator) {
            case '==': return $actual == $expected;
            case '!=': return $actual != $expected;
            case '>': return $actual > $expected;
            case '<': return $actual < $expected;
            case 'in': return in_array($actual, (array)$expected);
            default: return true;
        }
    }
    
    public function createWorkflow($data) {
        return $this->db->insert('mod_workflows', [
            'name' => $data['name'],
            'trigger_type' => $data['trigger_type'],
            'conditions' => json_encode($data['conditions']),
            'actions' => json_encode($data['actions']),
            'is_active' => 1,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_workflows (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    trigger_type VARCHAR(50),
    conditions JSON,
    actions JSON,
    is_active TINYINT(1) DEFAULT 1,
    created_at DATETIME
);

CREATE TABLE mod_workflow_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    workflow_id INT,
    context JSON,
    executed_at DATETIME
);
```

## Usage Examples
```php
$automation = new WorkflowAutomation();

// Trigger on event
$automation->triggerWorkflow('invoice_paid', [
    'invoice_id' => $invoiceId,
    'amount' => $amount
]);
```
