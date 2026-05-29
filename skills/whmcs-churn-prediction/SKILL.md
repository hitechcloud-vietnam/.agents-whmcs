# WHMCS Churn Prediction Skill

## Purpose
Provides patterns for implementing churn prediction in WHMCS, identifying at-risk customers, analyzing churn signals, and calculating churn probability scores.

## Implementation Patterns

### Churn Predictor
```php
<?php
class ChurnPredictor {
    private $db;
    
    public function calculateChurnRisk($clientId) {
        $signals = [];
        
        // Payment signals
        $paymentSignals = $this->analyzePaymentSignals($clientId);
        $signals = array_merge($signals, $paymentSignals);
        
        // Engagement signals
        $engagementSignals = $this->analyzeEngagementSignals($clientId);
        $signals = array_merge($signals, $engagementSignals);
        
        // Support signals
        $supportSignals = $this->analyzeSupportSignals($clientId);
        $signals = array_merge($signals, $supportSignals);
        
        return $this->calculateRiskScore($signals);
    }
    
    private function analyzePaymentSignals($clientId) {
        $signals = [];
        
        $failedPayments = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_payment_attempts
             WHERE client_id = ? AND status = 'failed' AND created_at > DATE_SUB(NOW(), INTERVAL 90 DAY)",
            [$clientId]
        )['count'];
        
        $signals[] = ['type' => 'failed_payments', 'value' => $failedPayments, 'weight' => 20];
        
        $latePayments = $this->db->select(
            "SELECT COUNT(*) as count FROM tblinvoices 
             WHERE userid = ? AND duedate < datepaid AND datepaid IS NOT NULL
             AND created_at > DATE_SUB(NOW(), INTERVAL 180 DAY)",
            [$clientId]
        )['count'];
        
        $signals[] = ['type' => 'late_payments', 'value' => $latePayments, 'weight' => 15];
        
        return $signals;
    }
    
    private function analyzeEngagementSignals($clientId) {
        $lastLogin = $this->db->select(
            "SELECT lastlogin FROM tblclients WHERE id = ?",
            [$clientId]
        )['lastlogin'];
        
        $daysSinceLogin = (time() - strtotime($lastLogin)) / 86400;
        
        $signals[] = [
            'type' => 'inactive_days',
            'value' => $daysSinceLogin,
            'weight' => $daysSinceLogin > 30 ? 30 : 10
        ];
        
        $activeServices = $this->db->select(
            "SELECT COUNT(*) as count FROM tblhosting WHERE userid = ? AND domainstatus = 'Active'",
            [$clientId]
        )['count'];
        
        $signals[] = ['type' => 'service_count', 'value' => $activeServices, 'weight' => -10];
        
        return $signals;
    }
    
    private function analyzeSupportSignals($clientId) {
        $openTickets = $this->db->select(
            "SELECT COUNT(*) as count FROM tbltickets 
             WHERE userid = ? AND status IN ('Open', 'Customer Reply')",
            [$clientId]
        )['count'];
        
        $signals[] = ['type' => 'open_tickets', 'value' => $openTickets, 'weight' => 10];
        
        return $signals;
    }
    
    private function calculateRiskScore($signals) {
        $totalScore = 0;
        $totalWeight = 0;
        
        foreach ($signals as $signal) {
            $normalizedValue = min(1, $signal['value'] / 10);
            $totalScore += $normalizedValue * $signal['weight'];
            $totalWeight += $signal['weight'];
        }
        
        $riskScore = $totalWeight > 0 ? ($totalScore / $totalWeight) * 100 : 0;
        
        return [
            'risk_score' => min(100, $riskScore),
            'risk_level' => $riskScore >= 70 ? 'high' : ($riskScore >= 40 ? 'medium' : 'low'),
            'signals' => $signals,
            'predicted_churn_probability' => $riskScore / 100
        ];
    }
    
    public function getAtRiskClients($threshold = 60) {
        return $this->db->select(
            "SELECT c.id, c.companyname, c.email,
                    (SELECT COUNT(*) FROM mod_churn_predictions WHERE client_id = c.id AND risk_score > ?) as risk_count
             FROM tblclients c
             WHERE EXISTS (SELECT 1 FROM mod_churn_predictions WHERE client_id = c.id AND risk_score > ?)
             ORDER BY risk_count DESC",
            [$threshold, $threshold]
        );
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_churn_predictions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    risk_score DECIMAL(5,2),
    risk_level VARCHAR(20),
    signals JSON,
    predicted_at DATETIME,
    actual_churned TINYINT(1) DEFAULT 0
);
```

## Usage Examples
```php
$predictor = new ChurnPredictor();
$risk = $predictor->calculateChurnRisk($clientId);

if ($risk['risk_level'] === 'high') {
    // Trigger retention actions
}
```
