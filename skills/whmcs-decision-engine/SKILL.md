# WHMCS Decision Engine Skill

## Purpose
Provides patterns for implementing a decision engine in WHMCS, making automated decisions based on multiple criteria, scoring systems, and policy rules.

## Implementation Patterns

### Decision Engine
```php
<?php
class DecisionEngine {
    private $db;
    
    public function makeDecision($context, $decisionType) {
        $decision = [
            'type' => $decisionType,
            'context' => $context,
            'timestamp' => date('Y-m-d H:i:s'),
            'options' => []
        ];
        
        switch ($decisionType) {
            case 'pricing':
                $decision = $this->evaluatePricing($context);
                break;
            case 'credit':
                $decision = $this->evaluateCredit($context);
                break;
            case 'approval':
                $decision = $this->evaluateApproval($context);
                break;
            case 'routing':
                $decision = $this->evaluateRouting($context);
                break;
        }
        
        $this->logDecision($decision);
        
        return $decision;
    }
    
    private function evaluatePricing($context) {
        $basePrice = $context['base_price'];
        $discount = 0;
        
        // Volume discount
        if ($context['quantity'] >= 10) {
            $discount += 5;
        }
        if ($context['quantity'] >= 100) {
            $discount += 10;
        }
        
        // Loyalty discount
        if ($context['client_tenure_months'] > 24) {
            $discount += 5;
        }
        
        // Contract discount
        if ($context['contract_length_months'] >= 12) {
            $discount += 8;
        }
        
        // Cap discount at 30%
        $discount = min($discount, 30);
        
        return [
            'final_price' => $basePrice * (1 - $discount / 100),
            'discount_applied' => $discount,
            'factors' => ['volume', 'loyalty', 'contract']
        ];
    }
    
    private function evaluateCredit($context) {
        $score = $context['credit_score'] ?? 500;
        $limit = 0;
        
        if ($score >= 850) $limit = 10000;
        elseif ($score >= 750) $limit = 5000;
        elseif ($score >= 650) $limit = 2500;
        else $limit = 500;
        
        return [
            'approved' => $score >= 500,
            'limit' => $limit,
            'score' => $score
        ];
    }
    
    private function evaluateApproval($context) {
        $riskScore = $context['risk_score'] ?? 0;
        
        if ($riskScore >= 80) {
            return ['decision' => 'deny', 'reason' => 'High risk score'];
        }
        if ($riskScore >= 50) {
            return ['decision' => 'review', 'reason' => 'Medium risk'];
        }
        return ['decision' => 'approve', 'reason' => 'Low risk'];
    }
    
    private function evaluateRouting($context) {
        $gateways = $context['available_gateways'];
        $selected = null;
        $score = 0;
        
        foreach ($gateways as $gateway) {
            $gatewayScore = $this->calculateGatewayScore($gateway, $context);
            if ($gatewayScore > $score) {
                $score = $gatewayScore;
                $selected = $gateway;
            }
        }
        
        return [
            'selected_gateway' => $selected,
            'score' => $score
        ];
    }
    
    private function calculateGatewayScore($gateway, $context) {
        $score = 100;
        
        // Success rate factor
        $score += ($gateway['success_rate'] - 90) * 2;
        
        // Cost factor
        $score -= $gateway['fee_percent'] * 5;
        
        return max(0, $score);
    }
    
    private function logDecision($decision) {
        $this->db->insert('mod_decision_log', [
            'type' => $decision['type'],
            'context' => json_encode($decision['context']),
            'result' => json_encode($decision),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_decision_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    type VARCHAR(50),
    context JSON,
    result JSON,
    created_at DATETIME
);

CREATE TABLE mod_decision_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255),
    decision_type VARCHAR(50),
    conditions JSON,
    scoring_weights JSON,
    thresholds JSON,
    is_active TINYINT(1) DEFAULT 1
);
```

## Usage Examples
```php
$engine = new DecisionEngine();

$decision = $engine->makeDecision([
    'credit_score' => 750,
    'requested_amount' => 2000
], 'credit');

if ($decision['approved']) {
    echo "Credit approved: $" . number_format($decision['limit']);
}
```
