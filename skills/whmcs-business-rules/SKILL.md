# WHMCS Business Rules Skill

## Purpose
Provides patterns for implementing a business rules engine in WHMCS, defining rule conditions, evaluating rules, and executing rule-based actions.

## Implementation Patterns

### Business Rules Engine
```php
<?php
class BusinessRulesEngine {
    private $db;
    
    public function evaluate($ruleSet, $context) {
        $rules = $this->db->select(
            "SELECT * FROM mod_business_rules WHERE ruleset = ? AND is_active = 1",
            [$ruleSet]
        );
        
        $results = [];
        
        foreach ($rules as $rule) {
            $results[] = $this->evaluateRule($rule, $context);
        }
        
        return $results;
    }
    
    private function evaluateRule($rule, $context) {
        $conditions = json_decode($rule['conditions'], true);
        $passed = true;
        
        foreach ($conditions as $condition) {
            $field = $condition['field'];
            $operator = $condition['operator'];
            $value = $condition['value'];
            
            $actual = $this->getNestedValue($context, $field);
            
            if (!$this->checkCondition($actual, $operator, $value)) {
                $passed = false;
                break;
            }
        }
        
        return [
            'rule_id' => $rule['id'],
            'name' => $rule['name'],
            'passed' => $passed,
            'action' => $passed ? $rule['action'] : null
        ];
    }
    
    private function checkCondition($actual, $operator, $expected) {
        switch ($operator) {
            case '==': return $actual == $expected;
            case '===': return $actual === $expected;
            case '!=': return $actual != $expected;
            case '>': return $actual > $expected;
            case '>=': return $actual >= $expected;
            case '<': return $actual < $expected;
            case '<=': return $actual <= $expected;
            case 'contains': return strpos($actual, $expected) !== false;
            case 'in': return in_array($actual, (array)$expected);
            default: return true;
        }
    }
    
    private function getNestedValue($context, $path) {
        $parts = explode('.', $path);
        $value = $context;
        foreach ($parts as $part) {
            $value = is_array($value) ? ($value[$part] ?? null) : null;
        }
        return $value;
    }
    
    public function executeAction($action, $context) {
        $actionData = json_decode($action, true);
        
        switch ($actionData['type']) {
            case 'block':
                return ['blocked' => true, 'reason' => $actionData['reason']];
            case 'require_approval':
                return ['requires_approval' => true, 'approvers' => $actionData['approvers']];
            case 'apply_discount':
                return ['discount' => $actionData['value']];
            default:
                return ['success' => true];
        }
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_business_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    ruleset VARCHAR(50),
    conditions JSON,
    action JSON,
    is_active TINYINT(1) DEFAULT 1,
    priority INT DEFAULT 50
);
```

## Usage Examples
```php
$engine = new BusinessRulesEngine();
$results = $engine->evaluate('pricing', [
    'client_tier' => 'gold',
    'order_amount' => 1000
]);

foreach ($results as $result) {
    if ($result['passed'] && $result['action']) {
        $engine->executeAction($result['action'], $context);
    }
}
```
