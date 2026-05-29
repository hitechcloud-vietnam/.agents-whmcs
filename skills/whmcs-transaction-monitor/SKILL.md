# WHMCS Transaction Monitor Skill

## Purpose
Provides patterns for monitoring transaction patterns, detecting anomalies, and managing financial transaction compliance in WHMCS.

## Implementation Patterns

### Transaction Monitor
```php
<?php
class TransactionMonitor {
    private $db;
    
    public function monitorTransaction($transaction) {
        $alerts = [];
        
        // Check for unusual patterns
        $velocityCheck = $this->checkVelocity($transaction);
        if ($velocityCheck['flagged']) {
            $alerts[] = $velocityCheck;
        }
        
        // Check amount patterns
        $amountCheck = $this->checkAmountPatterns($transaction);
        if ($amountCheck['flagged']) {
            $alerts[] = $amountCheck;
        }
        
        // Check frequency
        $frequencyCheck = $this->checkFrequency($transaction);
        if ($frequencyCheck['flagged']) {
            $alerts[] = $frequencyCheck;
        }
        
        // Log transaction
        $this->logTransaction($transaction, $alerts);
        
        return $alerts;
    }
    
    private function checkVelocity($tx) {
        $recentTx = $this->db->select(
            "SELECT COUNT(*) as count, SUM(amount) as total
             FROM mod_transaction_log
             WHERE client_id = ? AND created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)",
            [$tx['client_id']]
        );
        
        return [
            'flagged' => $recentTx['count'] > 5 || $recentTx['total'] > 5000,
            'type' => 'velocity',
            'count' => $recentTx['count'],
            'total' => $recentTx['total']
        ];
    }
    
    private function checkAmountPatterns($tx) {
        $structuringThreshold = 10000;
        
        return [
            'flagged' => $tx['amount'] >= 8000 && $tx['amount'] < $structuringThreshold,
            'type' => 'structuring',
            'amount' => $tx['amount']
        ];
    }
    
    private function checkFrequency($tx) {
        $avgInterval = $this->getAverageTransactionInterval($tx['client_id']);
        
        return [
            'flagged' => $avgInterval < 300,
            'type' => 'frequency',
            'avg_interval' => $avgInterval
        ];
    }
    
    private function logTransaction($tx, $alerts) {
        $this->db->insert('mod_transaction_log', [
            'client_id' => $tx['client_id'],
            'amount' => $tx['amount'],
            'type' => $tx['type'],
            'alerts' => json_encode($alerts),
            'flagged' => count($alerts) > 0,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_transaction_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    amount DECIMAL(10,2),
    type VARCHAR(50),
    alerts JSON,
    flagged TINYINT(1) DEFAULT 0,
    created_at DATETIME
);
```

## Usage Examples
```php
$monitor = new TransactionMonitor();
$alerts = $monitor->monitorTransaction([
    'client_id' => $clientId,
    'amount' => 9500,
    'type' => 'payment'
]);
```
