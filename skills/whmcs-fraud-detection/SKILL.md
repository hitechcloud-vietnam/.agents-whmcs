# WHMCS Fraud Detection Skill

## Purpose
Provides patterns for implementing fraud detection systems in WHMCS, analyzing transactions, identifying suspicious patterns, and preventing fraudulent activities.

## Implementation Patterns

### Fraud Detection Engine
```php
<?php
// includes/FraudDetection.class.php

class FraudDetectionEngine {
    private $db;
    private $rules;
    private $models;
    private $cache;
    
    public function __construct() {
        $this->db = console::db();
        $this->loadFraudRules();
        $this->loadModels();
    }
    
    private function loadFraudRules() {
        $this->rules = $this->db->select(
            "SELECT * FROM mod_fraud_rules WHERE is_active = 1"
        );
    }
    
    private function loadModels() {
        $this->models = [
            'velocity' => new VelocityCheckModel(),
            'geolocation' => new GeolocationModel(),
            'device' => new DeviceFingerprintModel(),
            'behavior' => new BehaviorAnalysisModel()
        ];
    }
    
    public function analyzeTransaction($transactionData) {
        $startTime = microtime(true);
        
        $analysis = [
            'transaction_id' => $transactionData['id'] ?? null,
            'timestamp' => date('Y-m-d H:i:s'),
            'risk_score' => 0,
            'flags' => [],
            'recommendations' => [],
            'checks_performed' => []
        ];
        
        foreach ($this->rules as $rule) {
            $checkResult = $this->evaluateRule($rule, $transactionData);
            $analysis['checks_performed'][] = $checkResult;
            
            if ($checkResult['triggered']) {
                $analysis['flags'][] = [
                    'rule' => $rule['name'],
                    'severity' => $rule['severity'],
                    'score' => $checkResult['score']
                ];
                $analysis['risk_score'] += $checkResult['score'];
            }
        }
        
        foreach ($this->models as $modelName => $model) {
            $modelResult = $model->evaluate($transactionData);
            if ($modelResult['triggered']) {
                $analysis['risk_score'] += $modelResult['score'];
            }
        }
        
        $analysis['risk_score'] = min(100, $analysis['risk_score']);
        $analysis['action'] = $analysis['risk_score'] >= 80 ? 'block' : ($analysis['risk_score'] >= 50 ? 'review' : 'allow');
        $analysis['processing_time_ms'] = (microtime(true) - $startTime) * 1000;
        
        return $analysis;
    }
    
    private function evaluateRule($rule, $transactionData) {
        $conditions = json_decode($rule['conditions'], true);
        $triggered = false;
        
        foreach ($conditions as $condition) {
            $field = $condition['field'];
            $operator = $condition['operator'];
            $value = $condition['value'];
            $actualValue = $this->getNestedValue($transactionData, $field);
            
            $passed = $this->checkCondition($actualValue, $operator, $value);
            if (!$passed && $condition['required'] ?? false) {
                $triggered = true;
            }
        }
        
        return [
            'rule' => $rule['name'],
            'triggered' => $triggered,
            'score' => $triggered ? $rule['score'] : 0
        ];
    }
    
    private function checkCondition($actual, $operator, $expected) {
        switch ($operator) {
            case '>': return $actual > $expected;
            case '>=': return $actual >= $expected;
            case '<': return $actual < $expected;
            case '==': return $actual == $expected;
            case 'in': return in_array($actual, (array)$expected);
            default: return true;
        }
    }
    
    private function getNestedValue($array, $path) {
        $parts = explode('.', $path);
        $value = $array;
        foreach ($parts as $part) {
            $value = is_array($value) ? ($value[$part] ?? null) : null;
        }
        return $value;
    }
}
```

### Velocity Check Model
```php
class VelocityCheckModel {
    public function evaluate($transactionData) {
        $score = 0;
        
        $orderCount = $this->db->select(
            "SELECT COUNT(*) as count FROM tblorders
             WHERE userid = ? AND datecreated > DATE_SUB(NOW(), INTERVAL 1 HOUR)",
            [$transactionData['client_id'] ?? 0]
        )['count'] ?? 0;
        
        if ($orderCount > 3) {
            $score += 20;
        }
        
        $totalAmount = $this->db->select(
            "SELECT SUM(amount) as total FROM mod_payment_transactions
             WHERE client_id = ? AND created_at > DATE_SUB(NOW(), INTERVAL 24 HOUR)",
            [$transactionData['client_id'] ?? 0]
        )['total'] ?? 0;
        
        if ($totalAmount > 1000) {
            $score += 15;
        }
        
        return [
            'model' => 'velocity',
            'score' => min(100, $score),
            'triggered' => $score > 30
        ];
    }
}
```

### Geolocation Model
```php
class GeolocationModel {
    public function evaluate($transactionData) {
        $score = 0;
        
        if (($transactionData['ip_country'] ?? '') !== ($transactionData['billing_country'] ?? '')) {
            $score += 30;
        }
        
        if ($transactionData['is_vpn'] ?? false) {
            $score += 25;
        }
        
        return [
            'model' => 'geolocation',
            'score' => min(100, $score),
            'triggered' => $score > 30
        ];
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_fraud_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    conditions JSON NOT NULL,
    score INT DEFAULT 10,
    severity ENUM('low', 'medium', 'high', 'critical') DEFAULT 'medium',
    is_active TINYINT(1) DEFAULT 1
);

CREATE TABLE mod_fraud_analysis (
    id INT AUTO_INCREMENT PRIMARY KEY,
    transaction_id INT,
    risk_score DECIMAL(5,2),
    action VARCHAR(20),
    flags JSON,
    created_at DATETIME
);

CREATE TABLE mod_device_fingerprints (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    fingerprint VARCHAR(255) NOT NULL,
    first_seen DATETIME,
    last_seen DATETIME
);
```

## Usage Examples

### Analyze Transaction
```php
$fraudEngine = new FraudDetectionEngine();
$analysis = $fraudEngine->analyzeTransaction([
    'id' => $orderId,
    'client_id' => $clientId,
    'amount' => 500,
    'ip_address' => $ip
]);

if ($analysis['action'] === 'block') {
    // Reject transaction
}
```
