# WHMCS Payment Routing Skill

## Purpose
Provides patterns for implementing smart payment routing in WHMCS, directing transactions through optimal payment gateways based on rules, costs, success rates, and customer preferences.

## Implementation Patterns

### Payment Router
```php
<?php
// includes/PaymentRouting.class.php

class PaymentRouter {
    private $db;
    private $gateways;
    private $rules;
    private $cache;
    
    public function __construct() {
        $this->db = console::db();
        $this->gateways = $this->loadGateways();
        $this->rules = $this->loadRules();
        $this->cache = console::cache();
    }
    
    private function loadGateways() {
        $cached = $this->cache->get('payment_gateways');
        if ($cached) return $cached;
        
        $gateways = $this->db->select(
            "SELECT * FROM mod_payment_gateways WHERE is_active = 1"
        );
        
        // Enrich with performance data
        foreach ($gateways as &$gateway) {
            $gateway['stats'] = $this->getGatewayStats($gateway['id']);
        }
        
        $this->cache->set('payment_gateways', $gateways, 300);
        return $gateways;
    }
    
    private function getGatewayStats($gatewayId) {
        $period = getenv('GATEWAY_STATS_PERIOD') ?: '30d';
        
        $stats = $this->db->select(
            "SELECT 
                COUNT(*) as total_transactions,
                SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END) as successful,
                SUM(CASE WHEN status = 'failed' THEN 1 ELSE 0 END) as failed,
                AVG(CASE WHEN status = 'success' THEN processing_time_ms END) as avg_time,
                SUM(CASE WHEN status = 'success' THEN amount END) as total_volume
             FROM mod_payment_transactions
             WHERE gateway_id = ? AND created_at > DATE_SUB(NOW(), INTERVAL ?)",
            [$gatewayId, $period]
        );
        
        return [
            'success_rate' => $stats['total_transactions'] > 0 
                ? ($stats['successful'] / $stats['total_transactions']) * 100 
                : 0,
            'avg_processing_time' => $stats['avg_time'] ?? 0,
            'total_volume' => $stats['total_volume'] ?? 0,
            'cost_rate' => $this->getGatewayCostRate($gatewayId),
            'volume_limit' => $this->getGatewayVolumeLimit($gatewayId),
            'daily_volume' => $this->getDailyVolume($gatewayId)
        ];
    }
    
    private function getGatewayCostRate($gatewayId) {
        $gateway = $this->db->select(
            "SELECT * FROM mod_payment_gateways WHERE id = ?",
            [$gatewayId]
        );
        
        return $gateway['transaction_fee_percent'] ?? 2.9;
    }
    
    // Route a payment to optimal gateway
    public function routePayment($context) {
        $candidateGateways = $this->evaluateGateways($context);
        
        if (empty($candidateGateways)) {
            throw new Exception("No available gateways for this transaction");
        }
        
        // Sort by score
        usort($candidateGateways, function($a, $b) {
            return $b['routing_score'] <=> $a['routing_score'];
        });
        
        // Select primary gateway
        $primary = $candidateGateways[0];
        
        // Prepare fallback chain
        $fallbacks = array_slice($candidateGateways, 1, 3);
        
        // Store routing decision
        $this->logRoutingDecision($context, $primary, $fallbacks);
        
        return [
            'primary_gateway' => $primary,
            'fallback_chain' => $fallbacks,
            'routing_reason' => $primary['routing_reason'],
            'estimated_cost' => $this->calculateTransactionCost($primary, $context['amount'])
        ];
    }
    
    private function evaluateGateways($context) {
        $candidates = [];
        
        foreach ($this->gateways as $gateway) {
            $evaluation = $this->evaluateGateway($gateway, $context);
            
            if ($evaluation['eligible']) {
                $candidates[] = $evaluation;
            }
        }
        
        return $candidates;
    }
    
    private function evaluateGateway($gateway, $context) {
        $score = 100;
        $reasons = [];
        
        // Check eligibility rules
        foreach ($this->rules as $rule) {
            if ($this->ruleApplies($rule, $context)) {
                $result = $this->applyRule($rule, $gateway, $context);
                $score += $result['score_adjustment'];
                
                if ($result['score_adjustment'] !== 0) {
                    $reasons[] = $result['reason'];
                }
                
                if (!$result['eligible']) {
                    return ['eligible' => false, 'reasons' => [$result['reason']]];
                }
            }
        }
        
        // Calculate success rate factor
        $successRateFactor = ($gateway['stats']['success_rate'] - 90) * 2;
        $score += $successRateFactor;
        
        // Calculate cost factor (lower cost = higher score)
        $costFactor = (3 - $gateway['stats']['cost_rate']) * 10;
        $score += $costFactor;
        
        // Check volume limits
        if ($gateway['stats']['daily_volume'] >= $gateway['stats']['volume_limit']) {
            return ['eligible' => false, 'reasons' => ['Daily volume limit reached']];
        }
        
        return [
            'eligible' => true,
            'gateway' => $gateway,
            'routing_score' => max(0, $score),
            'routing_reason' => implode('; ', $reasons)
        ];
    }
    
    private function ruleApplies($rule, $context) {
        // Check if rule conditions match context
        $conditions = $rule['conditions'] ?? [];
        
        foreach ($conditions as $condition) {
            if (!$this->checkCondition($condition, $context)) {
                return false;
            }
        }
        
        return true;
    }
    
    private function checkCondition($condition, $context) {
        $field = $condition['field'];
        $operator = $condition['operator'];
        $value = $condition['value'];
        
        $contextValue = $this->getNestedValue($context, $field);
        
        switch ($operator) {
            case '>':
                return $contextValue > $value;
            case '>=':
                return $contextValue >= $value;
            case '<':
                return $contextValue < $value;
            case '<=':
                return $contextValue <= $value;
            case '==':
                return $contextValue == $value;
            case 'in':
                return in_array($contextValue, (array)$value);
            case 'not_in':
                return !in_array($contextValue, (array)$value);
            default:
                return true;
        }
    }
    
    private function applyRule($rule, $gateway, $context) {
        $ruleType = $rule['type'];
        
        switch ($ruleType) {
            case 'cost_threshold':
                if ($gateway['stats']['cost_rate'] > $rule['threshold']) {
                    return ['eligible' => true, 'score_adjustment' => -20, 
                            'reason' => "High fee gateway: {$gateway['stats']['cost_rate']}%"];
                }
                return ['eligible' => true, 'score_adjustment' => 10, 'reason' => 'Low fee gateway'];
                
            case 'success_rate':
                if ($gateway['stats']['success_rate'] < $rule['min_rate']) {
                    return ['eligible' => false, 'score_adjustment' => 0, 
                            'reason' => "Low success rate: {$gateway['stats']['success_rate']}%"];
                }
                return ['eligible' => true, 'score_adjustment' => 5, 'reason' => 'High success rate'];
                
            case 'amount_limit':
                $minAmount = $rule['min_amount'] ?? 0;
                $maxAmount = $rule['max_amount'] ?? PHP_INT_MAX;
                if ($context['amount'] < $minAmount || $context['amount'] > $maxAmount) {
                    return ['eligible' => false, 'score_adjustment' => 0, 
                            'reason' => 'Amount outside gateway limits'];
                }
                return ['eligible' => true, 'score_adjustment' => 0, 'reason' => ''];
                
            case 'card_type':
                $allowedCardTypes = $rule['allowed_types'] ?? [];
                if (!empty($allowedCardTypes) && !in_array($context['card_type'], $allowedCardTypes)) {
                    return ['eligible' => false, 'score_adjustment' => 0, 
                            'reason' => "Card type not supported: {$context['card_type']}"];
                }
                return ['eligible' => true, 'score_adjustment' => 0, 'reason' => ''];
                
            case 'country':
                $allowedCountries = $rule['allowed_countries'] ?? [];
                if (!empty($allowedCountries) && !in_array($context['country'], $allowedCountries)) {
                    return ['eligible' => false, 'score_adjustment' => 0, 
                            'reason' => "Country not supported: {$context['country']}"];
                }
                return ['eligible' => true, 'score_adjustment' => 0, 'reason' => ''];
                
            default:
                return ['eligible' => true, 'score_adjustment' => 0, 'reason' => ''];
        }
    }
    
    // Process payment with fallback
    public function processWithFallback($context, $routingDecision) {
        $gateways = array_merge([$routingDecision['primary_gateway']], $routingDecision['fallback_chain']);
        
        foreach ($gateways as $gateway) {
            try {
                $result = $this->processPayment($gateway, $context);
                
                $this->logTransaction($gateway, $context, $result);
                
                return $result;
            } catch (Exception $e) {
                $this->logFailure($gateway, $context, $e);
                continue;
            }
        }
        
        throw new Exception("All payment gateways failed");
    }
    
    private function processPayment($gateway, $context) {
        $processor = $this->getProcessor($gateway['type']);
        
        return $processor->charge([
            'gateway_id' => $gateway['id'],
            'amount' => $context['amount'],
            'currency' => $context['currency'],
            'card_token' => $context['card_token'],
            'customer_id' => $context['customer_id']
        ]);
    }
}
```

### Routing Rule Manager
```php
class RoutingRuleManager {
    public function createRule($data) {
        $rule = [
            'name' => $data['name'],
            'type' => $data['type'],
            'conditions' => json_encode($data['conditions']),
            'priority' => $data['priority'] ?? 50,
            'action' => json_encode($data['action']),
            'is_active' => 1,
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        return $this->db->insert('mod_payment_routing_rules', $rule);
    }
    
    public function getRulesForContext($context) {
        $rules = $this->db->select(
            "SELECT * FROM mod_payment_routing_rules 
             WHERE is_active = 1 
             ORDER BY priority DESC"
        );
        
        return array_filter($rules, function($rule) use ($context) {
            return $this->ruleApplies($rule, $context);
        });
    }
    
    public function testRules($context) {
        $rules = $this->db->select(
            "SELECT * FROM mod_payment_routing_rules WHERE is_active = 1 ORDER BY priority DESC"
        );
        
        $results = [];
        
        foreach ($rules as $rule) {
            $conditions = json_decode($rule['conditions'], true);
            $matched = true;
            
            foreach ($conditions as $condition) {
                if (!$this->checkCondition($condition, $context)) {
                    $matched = false;
                    break;
                }
            }
            
            $results[] = [
                'rule' => $rule['name'],
                'type' => $rule['type'],
                'matched' => $matched,
                'action' => json_decode($rule['action'], true)
            ];
        }
        
        return $results;
    }
}
```

### Gateway Performance Monitor
```php
class GatewayPerformanceMonitor {
    public function monitorPerformance() {
        $alerts = [];
        
        foreach ($this->gateways as $gateway) {
            $stats = $this->getGatewayStats($gateway['id']);
            
            // Check success rate threshold
            if ($stats['success_rate'] < $gateway['min_success_rate']) {
                $alerts[] = [
                    'type' => 'low_success_rate',
                    'gateway' => $gateway['name'],
                    'severity' => 'high',
                    'message' => "Gateway {$gateway['name']} success rate dropped to {$stats['success_rate']}%",
                    'stats' => $stats
                ];
            }
            
            // Check response time threshold
            if ($stats['avg_processing_time'] > $gateway['max_processing_time']) {
                $alerts[] = [
                    'type' => 'slow_processing',
                    'gateway' => $gateway['name'],
                    'severity' => 'medium',
                    'message' => "Gateway {$gateway['name']} processing time: {$stats['avg_processing_time']}ms"
                ];
            }
            
            // Check volume limit
            $usagePct = ($stats['daily_volume'] / $gateway['daily_limit']) * 100;
            if ($usagePct > 80) {
                $alerts[] = [
                    'type' => 'volume_limit_warning',
                    'gateway' => $gateway['name'],
                    'severity' => $usagePct > 95 ? 'high' : 'medium',
                    'message' => "Gateway {$gateway['name']} at {$usagePct}% of daily limit"
                ];
            }
        }
        
        return $alerts;
    }
    
    public function calculateGatewayROI($gatewayId) {
        $stats = $this->getGatewayStats($gatewayId);
        $gateway = $this->getGateway($gatewayId);
        
        $monthlyCost = $this->calculateMonthlyCost($gatewayId);
        $monthlyVolume = $stats['total_volume'];
        
        return [
            'gateway' => $gateway['name'],
            'monthly_volume' => $monthlyVolume,
            'transaction_cost' => $monthlyCost,
            'cost_percentage' => $monthlyVolume > 0 ? ($monthlyCost / $monthlyVolume) * 100 : 0,
            'cost_per_transaction' => $stats['total_transactions'] > 0 
                ? $monthlyCost / $stats['total_transactions'] 
                : 0,
            'success_rate' => $stats['success_rate']
        ];
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_payment_gateways (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50) NOT NULL,
    is_active TINYINT(1) DEFAULT 1,
    config JSON,
    transaction_fee_percent DECIMAL(5,2) DEFAULT 2.9,
    transaction_fee_fixed DECIMAL(10,2) DEFAULT 0,
    daily_limit DECIMAL(15,2),
    monthly_limit DECIMAL(15,2),
    min_success_rate DECIMAL(5,2) DEFAULT 90,
    max_processing_time_ms INT DEFAULT 30000,
    supported_currencies JSON,
    supported_card_types JSON,
    supported_countries JSON,
    priority INT DEFAULT 50,
    created_at DATETIME,
    INDEX idx_active (is_active)
);

CREATE TABLE mod_payment_routing_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,
    conditions JSON,
    action JSON,
    priority INT DEFAULT 50,
    is_active TINYINT(1) DEFAULT 1,
    created_at DATETIME,
    INDEX idx_priority (priority)
);

CREATE TABLE mod_payment_transactions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    gateway_id INT NOT NULL,
    order_id INT,
    client_id INT,
    amount DECIMAL(15,2),
    currency VARCHAR(10),
    status ENUM('success', 'failed', 'pending', 'refunded', 'disputed'),
    processing_time_ms INT,
    error_code VARCHAR(50),
    error_message TEXT,
    routing_score DECIMAL(10,4),
    fallback_used TINYINT(1) DEFAULT 0,
    created_at DATETIME,
    INDEX idx_gateway (gateway_id),
    INDEX idx_status (status),
    INDEX idx_created (created_at)
);

CREATE TABLE mod_routing_decisions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    context_hash VARCHAR(64),
    context_data JSON,
    primary_gateway_id INT,
    fallback_chain JSON,
    routing_reason TEXT,
    estimated_cost DECIMAL(10,2),
    used TINYINT(1) DEFAULT 0,
    created_at DATETIME
);
```

## Usage Examples

### Route a Payment
```php
$router = new PaymentRouter();

$context = [
    'amount' => 500,
    'currency' => 'USD',
    'country' => 'US',
    'card_type' => 'visa',
    'customer_id' => $clientId,
    'order_id' => $orderId
];

$routingDecision = $router->routePayment($context);

echo "Primary Gateway: {$routingDecision['primary_gateway']['name']}\n";
echo "Routing Reason: {$routingDecision['routing_reason']}\n";
echo "Estimated Cost: $" . number_format($routingDecision['estimated_cost'], 2) . "\n";

// Process payment
$result = $router->processWithFallback($context, $routingDecision);
```

### Add Routing Rule
```php
$ruleManager = new RoutingRuleManager();

$ruleManager->createRule([
    'name' => 'High Value Priority',
    'type' => 'cost_threshold',
    'conditions' => [
        ['field' => 'amount', 'operator' => '>=', 'value' => 1000]
    ],
    'action' => [
        'boost_gateway' => 'stripe_premium',
        'score_adjustment' => 20
    ],
    'priority' => 100
]);
```

### Monitor Gateway Performance
```php
$monitor = new GatewayPerformanceMonitor();
$alerts = $monitor->monitorPerformance();

foreach ($alerts as $alert) {
    if ($alert['severity'] === 'high') {
        // Send urgent notification
        sendAlert($alert);
    }
}
```

## Best Practices

1. **Monitor continuously**: Track success rates and response times in real-time
2. **Use fallback chains**: Always have backup gateways for failed transactions
3. **Balance cost vs. reliability**: Sometimes higher fees are worth better success rates
4. **Set minimum thresholds**: Disable gateways that fall below acceptable success rates
5. **Track by card type**: Different card types may perform better on different gateways
6. **Consider geography**: Some gateways perform better in specific countries
7. **Load balance volume**: Distribute transactions to avoid hitting limits
8. **Log all decisions**: Maintain routing audit trail for troubleshooting and optimization