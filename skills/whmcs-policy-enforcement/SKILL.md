# WHMCS Policy Enforcement Skill

## Purpose
Provides patterns for implementing policy enforcement in WHMCS, defining business rules, validating actions against policies, and enforcing compliance with organizational standards.

## Implementation Patterns

### Policy Engine
```php
<?php
// includes/PolicyEnforcement.class.php

class PolicyEngine {
    private $db;
    private $cache;
    private $policies = [];
    
    public function __construct() {
        $this->db = console::db();
        $this->cache = console::cache();
        $this->loadPolicies();
    }
    
    private function loadPolicies() {
        $cacheKey = 'policies_active';
        $cached = $this->cache->get($cacheKey);
        
        if ($cached) {
            $this->policies = $cached;
            return;
        }
        
        $policies = $this->db->select(
            "SELECT * FROM mod_policies WHERE is_active = 1"
        );
        
        foreach ($policies as $policy) {
            $this->policies[$policy['code']] = [
                'id' => $policy['id'],
                'name' => $policy['name'],
                'description' => $policy['description'],
                'rules' => json_decode($policy['rules'], true),
                'enforcement_level' => $policy['enforcement_level'],
                'actions' => json_decode($policy['actions'], true),
                'exemptions' => json_decode($policy['exemptions'], true)
            ];
        }
        
        $this->cache->set($cacheKey, $this->policies, 3600);
    }
    
    // Create a new policy
    public function createPolicy($data) {
        $policy = [
            'code' => $data['code'],
            'name' => $data['name'],
            'description' => $data['description'],
            'rules' => json_encode($data['rules']),
            'enforcement_level' => $data['enforcement_level'] ?? 'strict',
            'actions' => json_encode($data['actions'] ?? []),
            'exemptions' => json_encode($data['exemptions'] ?? []),
            'is_active' => 1,
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        $id = $this->db->insert('mod_policies', $policy);
        $this->invalidateCache();
        
        return $id;
    }
    
    // Enforce a policy
    public function enforce($policyCode, $context) {
        $policy = $this->policies[$policyCode] ?? null;
        
        if (!$policy) {
            throw new Exception("Policy not found: {$policyCode}");
        }
        
        // Check exemptions
        if ($this->isExempt($policy, $context)) {
            return [
                'allowed' => true,
                'policy' => $policyCode,
                'reason' => 'exempt'
            ];
        }
        
        // Evaluate rules
        $result = $this->evaluateRules($policy['rules'], $context);
        
        if ($result['allowed']) {
            return [
                'allowed' => true,
                'policy' => $policyCode,
                'rules_met' => $result['met_rules'],
                'reason' => 'all_rules_passed'
            ];
        }
        
        // Handle violation
        $this->handleViolation($policy, $context, $result);
        
        return [
            'allowed' => false,
            'policy' => $policyCode,
            'violations' => $result['violations'],
            'actions_taken' => $this->executeActions($policy['actions'], $context)
        ];
    }
    
    private function isExempt($policy, $context) {
        $exemptions = $policy['exemptions'];
        
        foreach ($exemptions as $exemption) {
            if ($this->matchesExemption($exemption, $context)) {
                return true;
            }
        }
        
        return false;
    }
    
    private function matchesExemption($exemption, $context) {
        switch ($exemption['type']) {
            case 'role':
                return in_array($context['user_role'], $exemption['values'] ?? []);
                
            case 'ip_whitelist':
                return $this->isIPWhitelisted($context['ip'], $exemption['values'] ?? []);
                
            case 'user':
                return in_array($context['user_id'], $exemption['values'] ?? []);
                
            case 'time_window':
                return $this->isWithinTimeWindow($exemption['start'], $exemption['end']);
                
            case 'approval':
                return $this->hasApproval($exemption['approval_type'], $context);
                
            default:
                return false;
        }
    }
    
    private function evaluateRules($rules, $context) {
        $result = [
            'allowed' => true,
            'violations' => [],
            'met_rules' => []
        ];
        
        foreach ($rules as $rule) {
            $evaluation = $this->evaluateRule($rule, $context);
            
            if ($evaluation['passed']) {
                $result['met_rules'][] = $rule;
            } else {
                $result['violations'][] = $evaluation;
                
                if ($rule['required']) {
                    $result['allowed'] = false;
                }
            }
        }
        
        return $result;
    }
    
    private function evaluateRule($rule, $context) {
        $condition = $rule['condition'];
        $expected = $rule['expected'];
        
        $actual = $this->resolveValue($condition['field'], $context);
        
        $passed = $this->compareValues($actual, $condition['operator'], $expected);
        
        return [
            'rule' => $rule,
            'expected' => $expected,
            'actual' => $actual,
            'operator' => $condition['operator'],
            'passed' => $passed,
            'message' => sprintf($rule['message'] ?? 'Rule check failed', $actual, $expected)
        ];
    }
    
    private function resolveValue($field, $context) {
        // Support nested field resolution (e.g., "user.credit_limit")
        $parts = explode('.', $field);
        $value = $context;
        
        foreach ($parts as $part) {
            if (is_array($value) && isset($value[$part])) {
                $value = $value[$part];
            } else {
                return null;
            }
        }
        
        return $value;
    }
    
    private function compareValues($actual, $operator, $expected) {
        switch ($operator) {
            case '==':
                return $actual == $expected;
            case '===':
                return $actual === $expected;
            case '!=':
                return $actual != $expected;
            case '>':
                return $actual > $expected;
            case '>=':
                return $actual >= $expected;
            case '<':
                return $actual < $expected;
            case '<=':
                return $actual <= $expected;
            case 'in':
                return in_array($actual, (array)$expected);
            case 'not_in':
                return !in_array($actual, (array)$expected);
            case 'contains':
                return strpos($actual, $expected) !== false;
            case 'matches':
                return preg_match($expected, $actual) === 1;
            default:
                return false;
        }
    }
    
    private function handleViolation($policy, $context, $result) {
        $violation = [
            'policy_id' => $policy['id'],
            'policy_code' => $policy['name'],
            'context' => json_encode($context),
            'violations' => json_encode($result['violations']),
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        $this->db->insert('mod_policy_violations', $violation);
        
        // Log for audit
        logActivity("Policy violation: {$policy['name']}", 'warning');
    }
    
    private function executeActions($actions, $context) {
        $executed = [];
        
        foreach ($actions as $action) {
            $result = $this->executeAction($action, $context);
            $executed[] = $result;
        }
        
        return $executed;
    }
    
    private function executeAction($action, $context) {
        switch ($action['type']) {
            case 'notify':
                return $this->sendNotification($action, $context);
                
            case 'log':
                return $this->logAction($action, $context);
                
            case 'block':
                return $this->blockAction($context);
                
            case 'require_approval':
                return $this->requestApproval($action, $context);
                
            case 'modify':
                return $this->modifyRequest($action, $context);
                
            default:
                return ['type' => $action['type'], 'status' => 'unknown_action'];
        }
    }
}
```

### Common Policy Definitions
```php
class PolicyDefinitions {
    // Credit limit policy
    public static function creditLimitPolicy($limit) {
        return [
            'code' => 'credit_limit',
            'name' => 'Credit Limit Policy',
            'description' => 'Enforces credit limit restrictions on orders',
            'enforcement_level' => 'strict',
            'rules' => [
                [
                    'name' => 'order_amount_within_limit',
                    'condition' => ['field' => 'order.total', 'operator' => '<=', 'value' => $limit],
                    'required' => true,
                    'message' => 'Order amount exceeds credit limit'
                ]
            ],
            'actions' => [
                ['type' => 'block', 'message' => 'Credit limit exceeded'],
                ['type' => 'notify', 'recipients' => ['admin'], 'message' => 'Credit limit violation']
            ]
        ];
    }
    
    // Minimum payment policy
    public static function minimumPaymentPolicy($amount) {
        return [
            'code' => 'minimum_payment',
            'name' => 'Minimum Payment Policy',
            'description' => 'Requires minimum payment amounts',
            'enforcement_level' => 'strict',
            'rules' => [
                [
                    'name' => 'payment_amount_minimum',
                    'condition' => ['field' => 'payment.amount', 'operator' => '>=', 'value' => $amount],
                    'required' => true,
                    'message' => "Payment amount must be at least {$amount}"
                ]
            ]
        ];
    }
    
    // Service provisioning policy
    public static function provisioningPolicy($config = []) {
        return [
            'code' => 'service_provisioning',
            'name' => 'Service Provisioning Policy',
            'description' => 'Controls service provisioning based on various conditions',
            'enforcement_level' => 'strict',
            'rules' => [
                [
                    'name' => 'no_overdue_invoices',
                    'condition' => ['field' => 'client.overdue_count', 'operator' => '==', 'value' => 0],
                    'required' => true,
                    'message' => 'Client has overdue invoices'
                ],
                [
                    'name' => 'valid_payment_method',
                    'condition' => ['field' => 'payment.method', 'operator' => 'in', 'value' => ['credit_card', 'bank_transfer', 'paypal']],
                    'required' => true,
                    'message' => 'No valid payment method on file'
                ],
                [
                    'name' => 'fraud_check_passed',
                    'condition' => ['field' => 'fraud.score', 'operator' => '<', 'value' => $config['fraud_threshold'] ?? 80],
                    'required' => true,
                    'message' => 'Fraud check failed'
                ]
            ],
            'actions' => [
                ['type' => 'log', 'message' => 'Service provisioning blocked'],
                ['type' => 'notify', 'recipients' => ['admin'], 'message' => 'Provisioning blocked']
            ],
            'exemptions' => [
                ['type' => 'role', 'values' => ['admin', 'support']]
            ]
        ];
    }
    
    // Data retention policy
    public static function dataRetentionPolicy($days) {
        return [
            'code' => 'data_retention',
            'name' => 'Data Retention Policy',
            'description' => 'Enforces data retention requirements',
            'enforcement_level' => 'strict',
            'rules' => [
                [
                    'name' => 'within_retention_period',
                    'condition' => ['field' => 'data.age_days', 'operator' => '<=', 'value' => $days],
                    'required' => true,
                    'message' => 'Data exceeds retention period'
                ]
            ]
        ];
    }
    
    // Pricing policy
    public static function pricingPolicy($config = []) {
        return [
            'code' => 'pricing',
            'name' => 'Pricing Policy',
            'description' => 'Enforces pricing rules and discounts',
            'enforcement_level' => 'flexible',
            'rules' => [
                [
                    'name' => 'discount_within_limits',
                    'condition' => ['field' => 'order.discount_pct', 'operator' => '<=', 'value' => $config['max_discount'] ?? 20],
                    'required' => false,
                    'message' => 'Discount exceeds allowed limit'
                ],
                [
                    'name' => 'price_not_below_cost',
                    'condition' => ['field' => 'order.total', 'operator' => '>=', 'value' => 0],
                    'required' => true,
                    'message' => 'Price cannot be below zero'
                ]
            ],
            'actions' => [
                ['type' => 'require_approval', 'approvers' => ['manager'], 'message' => 'Discount requires approval']
            ],
            'exemptions' => [
                ['type' => 'role', 'values' => ['admin', 'sales_manager']]
            ]
        ];
    }
}
```

### Policy Manager
```php
class PolicyManager {
    public function enforceAll($context, $category = null) {
        $policies = $this->getPoliciesForContext($context, $category);
        $results = [];
        
        foreach ($policies as $policy) {
            $result = $this->enforce($policy['code'], $context);
            $results[$policy['code']] = $result;
            
            if (!$result['allowed']) {
                return [
                    'allowed' => false,
                    'violations' => $results
                ];
            }
        }
        
        return [
            'allowed' => true,
            'results' => $results
        ];
    }
    
    private function getPoliciesForContext($context, $category) {
        $query = "SELECT * FROM mod_policies WHERE is_active = 1";
        $params = [];
        
        if ($category) {
            $query .= " AND category = ?";
            $params[] = $category;
        }
        
        $policies = $this->db->select($query, $params);
        
        // Filter by scope
        return array_filter($policies, function($policy) use ($context) {
            $scope = json_decode($policy['scope'], true) ?? [];
            
            // If no scope defined, apply to all
            if (empty($scope)) {
                return true;
            }
            
            // Check if context matches scope
            foreach ($scope as $rule) {
                if ($this->matchesScope($rule, $context)) {
                    return true;
                }
            }
            
            return false;
        });
    }
    
    private function matchesScope($rule, $context) {
        $field = $rule['field'];
        $values = $rule['values'];
        
        $actual = $context[$field] ?? null;
        
        return in_array($actual, $values);
    }
    
    public function requestExemption($policyCode, $context, $reason) {
        $request = [
            'policy_code' => $policyCode,
            'context' => json_encode($context),
            'reason' => $reason,
            'requested_by' => $context['user_id'],
            'requested_at' => date('Y-m-d H:i:s'),
            'status' => 'pending'
        ];
        
        $id = $this->db->insert('mod_policy_exemption_requests', $request);
        
        // Notify approvers
        $this->notifyApprovers($id, $policyCode);
        
        return $id;
    }
    
    public function approveExemption($requestId, $approverId, $validUntil = null) {
        $request = $this->getRequest($requestId);
        
        $approval = [
            'request_id' => $requestId,
            'approved_by' => $approverId,
            'approved_at' => date('Y-m-d H:i:s'),
            'valid_until' => $validUntil,
            'conditions' => null
        ];
        
        $this->db->insert('mod_policy_exemption_approvals', $approval);
        $this->db->where('id', $requestId)
            ->update('mod_policy_exemption_requests', ['status' => 'approved']);
        
        return true;
    }
}
```

### Policy Reporting
```php
class PolicyReporting {
    public function getComplianceReport($period = '30d') {
        return [
            'summary' => $this->getSummaryStats($period),
            'violations' => $this->getViolationReport($period),
            'exemptions' => $this->getExemptionReport($period),
            'policy_effectiveness' => $this->getPolicyEffectiveness($period),
            'recommendations' => $this->generateRecommendations()
        ];
    }
    
    private function getSummaryStats($period) {
        return $this->db->select(
            "SELECT 
                COUNT(DISTINCT policy_id) as total_policies,
                COUNT(*) as total_enforcements,
                COUNT(CASE WHEN allowed = 1 THEN 1 END) as allowed_actions,
                COUNT(CASE WHEN allowed = 0 THEN 1 END) as blocked_actions,
                COUNT(DISTINCT context->>'$.user_id') as affected_users
             FROM mod_policy_enforcements
             WHERE created_at > DATE_SUB(NOW(), INTERVAL ?)",
            [$period]
        );
    }
    
    private function getViolationReport($period) {
        return $this->db->select(
            "SELECT 
                policy_code,
                COUNT(*) as violation_count,
                AVG(violations->>'$.score') as avg_severity,
                GROUP_CONCAT(DISTINCT context->>'$.user_role') as affected_roles
             FROM mod_policy_violations
             WHERE created_at > DATE_SUB(NOW(), INTERVAL ?)
             GROUP BY policy_code
             ORDER BY violation_count DESC",
            [$period]
        );
    }
    
    private function getPolicyEffectiveness($period) {
        $policies = $this->db->select(
            "SELECT * FROM mod_policies WHERE is_active = 1"
        );
        
        $effectiveness = [];
        foreach ($policies as $policy) {
            $stats = $this->getPolicyStats($policy['code'], $period);
            
            $effectiveness[] = [
                'policy' => $policy['name'],
                'code' => $policy['code'],
                'enforcement_count' => $stats['total'],
                'violation_rate' => $stats['total'] > 0 
                    ? ($stats['violations'] / $stats['total']) * 100 
                    : 0,
                'avg_resolution_time' => $stats['avg_resolution'],
                'effectiveness_score' => $this->calculateEffectivenessScore($stats)
            ];
        }
        
        return $effectiveness;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_policies (
    id INT AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(50),
    rules JSON NOT NULL,
    actions JSON,
    exemptions JSON,
    scope JSON,
    enforcement_level ENUM('strict', 'flexible', 'advisory') DEFAULT 'strict',
    is_active TINYINT(1) DEFAULT 1,
    created_at DATETIME,
    updated_at DATETIME
);

CREATE TABLE mod_policy_enforcements (
    id INT AUTO_INCREMENT PRIMARY KEY,
    policy_id INT NOT NULL,
    policy_code VARCHAR(100) NOT NULL,
    context JSON,
    allowed TINYINT(1),
    violations JSON,
    actions_taken JSON,
    created_at DATETIME,
    INDEX idx_policy (policy_id),
    INDEX idx_allowed (allowed),
    INDEX idx_created (created_at)
);

CREATE TABLE mod_policy_violations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    policy_id INT NOT NULL,
    policy_code VARCHAR(100) NOT NULL,
    context JSON,
    violations JSON,
    resolved TINYINT(1) DEFAULT 0,
    resolved_by INT,
    resolved_at DATETIME,
    created_at DATETIME,
    INDEX idx_policy (policy_id),
    INDEX idx_resolved (resolved)
);

CREATE TABLE mod_policy_exemption_requests (
    id INT AUTO_INCREMENT PRIMARY KEY,
    policy_code VARCHAR(100) NOT NULL,
    context JSON,
    reason TEXT,
    requested_by INT NOT NULL,
    requested_at DATETIME,
    status ENUM('pending', 'approved', 'denied') DEFAULT 'pending',
    decided_by INT,
    decided_at DATETIME,
    decision_note TEXT,
    INDEX idx_status (status)
);

CREATE TABLE mod_policy_exemption_approvals (
    id INT AUTO_INCREMENT PRIMARY KEY,
    request_id INT NOT NULL,
    approved_by INT NOT NULL,
    approved_at DATETIME,
    valid_until DATETIME,
    conditions JSON
);

CREATE TABLE mod_policy_ip_whitelist (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ip_address VARCHAR(45) NOT NULL,
    description VARCHAR(255),
    added_by INT,
    added_at DATETIME,
    expires_at DATETIME
);
```

## Usage Examples

### Enforce Credit Limit
```php
$engine = new PolicyEngine();
$result = $engine->enforce('credit_limit', [
    'order' => ['total' => 1500],
    'user_id' => $clientId,
    'user_role' => 'customer'
]);

if (!$result['allowed']) {
    echo "Order blocked: {$result['violations'][0]['message']}\n";
}
```

### Check All Policies for Order
```php
$manager = new PolicyManager();
$result = $manager->enforceAll([
    'order' => $orderData,
    'client' => $clientData,
    'user_id' => $adminId,
    'user_role' => 'admin'
], 'orders');

if (!$result['allowed']) {
    // Handle violations
}
```

### Request Policy Exemption
```php
$manager = new PolicyManager();
$requestId = $manager->requestExemption('discount_limit', [
    'user_id' => $clientId,
    'order_id' => $orderId
], 'End of quarter sale promotion');

// Notify managers to approve
```

## Best Practices

1. **Start with advisory**: Test policies in advisory mode before enforcement
2. **Use clear naming**: Policy codes should be descriptive and consistent
3. **Document exemptions**: Track all policy exemptions for audit
4. **Monitor effectiveness**: Track violation rates and adjust rules
5. **Implement hierarchies**: Some policies should override others
6. **Provide feedback**: Tell users why their action was blocked
7. **Regular reviews**: Review policy effectiveness quarterly
8. **Test thoroughly**: Validate rules against various scenarios