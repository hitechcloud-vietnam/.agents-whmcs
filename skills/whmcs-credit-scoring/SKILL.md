# WHMCS Credit Scoring Skill

## Purpose
Provides patterns for implementing credit scoring systems for WHMCS clients, evaluating creditworthiness, managing credit limits, and automating credit decisions.

## Implementation Patterns

### Credit Score Calculator
```php
<?php
// includes/CreditScoring.class.php

class CreditScoringSystem {
    private $db;
    private $weights;
    private $thresholds;
    
    public function __construct() {
        $this->db = console::db();
        $this->initializeWeights();
        $this->initializeThresholds();
    }
    
    private function initializeWeights() {
        $this->weights = [
            'payment_history' => 0.35,
            'account_age' => 0.15,
            'credit_utilization' => 0.20,
            'payment_frequency' => 0.10,
            'dispute_ratio' => 0.08,
            'account_activity' => 0.07,
            'risk_indicators' => 0.05
        ];
    }
    
    private function initializeThresholds() {
        $this->thresholds = [
            'excellent' => 850,
            'very_good' => 750,
            'good' => 700,
            'fair' => 650,
            'poor' => 550,
            'very_poor' => 0
        ];
    }
    
    // Calculate credit score for a client
    public function calculateScore($clientId) {
        $factors = [
            'payment_history' => $this->evaluatePaymentHistory($clientId),
            'account_age' => $this->evaluateAccountAge($clientId),
            'credit_utilization' => $this->evaluateCreditUtilization($clientId),
            'payment_frequency' => $this->evaluatePaymentFrequency($clientId),
            'dispute_ratio' => $this->evaluateDisputeRatio($clientId),
            'account_activity' => $this->evaluateAccountActivity($clientId),
            'risk_indicators' => $this->evaluateRiskIndicators($clientId)
        ];
        
        $weightedScore = 0;
        $totalWeight = 0;
        
        foreach ($factors as $factor => $score) {
            $weightedScore += $score * $this->weights[$factor];
            $totalWeight += $this->weights[$factor];
        }
        
        // Normalize to 0-1000 scale
        $rawScore = ($weightedScore / $totalWeight) * 1000;
        $finalScore = $this->applyModifiers($clientId, $rawScore);
        
        // Cache the score
        $this->cacheScore($clientId, $finalScore, $factors);
        
        return [
            'score' => round($finalScore),
            'grade' => $this->getGrade($finalScore),
            'factors' => $factors,
            'recommendations' => $this->getRecommendations($finalScore, $factors),
            'calculated_at' => date('Y-m-d H:i:s')
        ];
    }
    
    private function evaluatePaymentHistory($clientId) {
        $payments = $this->db->select(
            "SELECT 
                COUNT(*) as total_payments,
                SUM(CASE WHEN paid = 1 THEN 1 ELSE 0 END) as on_time,
                SUM(CASE WHEN paid = 0 AND duedate < NOW() THEN 1 ELSE 0 END) as overdue,
                AVG(DATEDIFF(datepaid, duedate)) as avg_delay
             FROM tblinvoices
             WHERE userid = ?",
            [$clientId]
        );
        
        if ($payments['total_payments'] == 0) return 500; // Neutral
        
        $onTimeRate = $payments['total_payments'] > 0 
            ? $payments['on_time'] / $payments['total_payments'] 
            : 0;
        
        // Calculate score based on payment performance
        $baseScore = $onTimeRate * 600; // Up to 600 points
        $latePenalty = min($payments['overdue'] * 20, 200); // Up to 200 point penalty
        $delayBonus = max(0, 100 - ($payments['avg_delay'] * 10)); // Up to 100 bonus
        
        return max(0, min(1000, $baseScore - $latePenalty + $delayBonus));
    }
    
    private function evaluateAccountAge($clientId) {
        $result = $this->db->select(
            "SELECT DATEDIFF(NOW(), regdate) as age_days FROM tblclients WHERE id = ?",
            [$clientId]
        );
        
        $days = $result['age_days'] ?? 0;
        
        if ($days >= 365 * 5) return 1000; // 5+ years
        if ($days >= 365 * 3) return 900;  // 3+ years
        if ($days >= 365 * 2) return 800;  // 2+ years
        if ($days >= 365 * 1) return 700;  // 1+ years
        if ($days >= 180) return 600;      // 6+ months
        if ($days >= 90) return 500;       // 3+ months
        return max(100, $days * 3);       // Scale by days
    }
    
    private function evaluateCreditUtilization($clientId) {
        $creditLimit = $this->getCreditLimit($clientId);
        if ($creditLimit <= 0) return 500;
        
        $currentBalance = $this->getCurrentBalance($clientId);
        $utilization = ($currentBalance / $creditLimit) * 100;
        
        if ($utilization <= 10) return 1000;
        if ($utilization <= 30) return 900;
        if ($utilization <= 50) return 700;
        if ($utilization <= 70) return 500;
        if ($utilization <= 90) return 300;
        return 100; // Over 90%
    }
    
    private function evaluatePaymentFrequency($clientId) {
        $payments = $this->db->select(
            "SELECT COUNT(DISTINCT DATE(datepaid)) as payment_days
             FROM tblaccounts
             WHERE userid = ? AND datepaid > DATE_SUB(NOW(), INTERVAL 365 DAY)",
            [$clientId]
        );
        
        $days = $payments['payment_days'] ?? 0;
        
        if ($days >= 300) return 1000;
        if ($days >= 200) return 850;
        if ($days >= 100) return 700;
        if ($days >= 50) return 550;
        return max(100, $days * 10);
    }
    
    private function evaluateDisputeRatio($clientId) {
        $tickets = $this->db->select(
            "SELECT 
                COUNT(*) as total_tickets,
                SUM(CASE WHEN status = 'Closed' AND (subject LIKE '%dispute%' OR subject LIKE '%chargeback%') THEN 1 ELSE 0 END) as disputes
             FROM tbltickets
             WHERE userid = ?",
            [$clientId]
        );
        
        if ($tickets['total_tickets'] == 0) return 500;
        
        $disputeRatio = $tickets['disputes'] / $tickets['total_tickets'];
        
        if ($disputeRatio == 0) return 1000;
        if ($disputeRatio <= 0.01) return 850;
        if ($disputeRatio <= 0.03) return 700;
        if ($disputeRatio <= 0.05) return 500;
        if ($disputeRatio <= 0.10) return 300;
        return 100;
    }
    
    private function evaluateAccountActivity($clientId) {
        $services = $this->db->select(
            "SELECT COUNT(*) as active_services,
                    SUM(monthly_amount) as monthly_value
             FROM tblhosting
             WHERE userid = ? AND domainstatus = 'Active'",
            [$clientId]
        );
        
        $activityScore = min(500, $services['active_services'] * 50);
        $valueScore = min(500, $services['monthly_value'] / 10);
        
        return $activityScore + $valueScore;
    }
    
    private function evaluateRiskIndicators($clientId) {
        $riskScore = 500; // Start neutral
        
        // Check for red flags
        $redFlags = $this->getRiskIndicators($clientId);
        
        foreach ($redFlags as $flag) {
            $riskScore -= $flag['severity'];
        }
        
        // Check for green flags
        $greenFlags = $this->getPositiveIndicators($clientId);
        foreach ($greenFlags as $flag) {
            $riskScore += $flag['weight'];
        }
        
        return max(0, min(1000, $riskScore));
    }
    
    private function applyModifiers($clientId, $baseScore) {
        $modifiers = $this->db->select(
            "SELECT * FROM mod_credit_modifiers WHERE client_id = ? AND active = 1",
            [$clientId]
        );
        
        foreach ($modifiers as $modifier) {
            $baseScore *= (1 + $modifier['multiplier']);
            $baseScore += $modifier['adjustment'];
        }
        
        return max(0, min(1000, $baseScore));
    }
    
    public function getGrade($score) {
        foreach ($this->thresholds as $grade => $threshold) {
            if ($score >= $threshold) {
                return $grade;
            }
        }
        return 'very_poor';
    }
    
    private function getRecommendations($score, $factors) {
        $recommendations = [];
        
        foreach ($factors as $factor => $value) {
            if ($value < 500) {
                $recommendations[] = [
                    'factor' => $factor,
                    'current' => $value,
                    'suggestion' => $this->getFactorSuggestion($factor, $value)
                ];
            }
        }
        
        return $recommendations;
    }
}
```

### Credit Limit Manager
```php
class CreditLimitManager {
    public function calculateCreditLimit($clientId) {
        $score = $this->calculateScore($clientId);
        
        // Base limits by grade
        $baseLimits = [
            'excellent' => 10000,
            'very_good' => 5000,
            'good' => 2500,
            'fair' => 1000,
            'poor' => 500,
            'very_poor' => 0
        ];
        
        $baseLimit = $baseLimits[$score['grade']] ?? 0;
        
        // Adjust based on payment history
        $paymentMultiplier = $this->getPaymentMultiplier($clientId);
        
        // Adjust based on monthly revenue
        $revenueMultiplier = $this->getRevenueMultiplier($clientId);
        
        $finalLimit = $baseLimit * $paymentMultiplier * $revenueMultiplier;
        
        // Cap at maximum
        $maxLimit = $this->getMaxCreditLimit();
        $finalLimit = min($finalLimit, $maxLimit);
        
        // Floor at minimum
        $finalLimit = max($finalLimit, $this->getMinCreditLimit());
        
        return round($finalLimit, 2);
    }
    
    public function updateCreditLimit($clientId, $newLimit, $reason) {
        $oldLimit = $this->getCreditLimit($clientId);
        
        $this->db->where('client_id', $clientId)
            ->update('mod_credit_limits', [
                'limit_amount' => $newLimit,
                'updated_at' => date('Y-m-d H:i:s'),
                'updated_by' => $this->getCurrentUserId()
            ]);
        
        $this->logLimitChange($clientId, $oldLimit, $newLimit, $reason);
        
        return $newLimit;
    }
    
    public function checkCreditAvailability($clientId, $amount) {
        $limit = $this->getCreditLimit($clientId);
        $balance = $this->getCurrentBalance($clientId);
        $available = $limit - $balance;
        
        return [
            'available' => $available >= $amount,
            'limit' => $limit,
            'balance' => $balance,
            'available_credit' => $available,
            'requested' => $amount,
            'shortfall' => $amount > $available ? $amount - $available : 0
        ];
    }
    
    public function reserveCredit($clientId, $amount, $description) {
        $check = $this->checkCreditAvailability($clientId, $amount);
        if (!$check['available']) {
            return false;
        }
        
        return $this->db->insert('mod_credit_reservations', [
            'client_id' => $clientId,
            'amount' => $amount,
            'description' => $description,
            'reserved_at' => date('Y-m-d H:i:s'),
            'status' => 'active'
        ]);
    }
    
    public function releaseCredit($reservationId) {
        $reservation = $this->getReservation($reservationId);
        
        $this->db->where('id', $reservationId)
            ->update('mod_credit_reservations', [
                'status' => 'released',
                'released_at' => date('Y-m-d H:i:s')
            ]);
        
        return true;
    }
}
```

### Automated Credit Decision Engine
```php
class CreditDecisionEngine {
    public function makeDecision($clientId, $requestType, $amount) {
        $score = $this->calculateScore($clientId);
        $history = $this->getRequestHistory($clientId);
        
        $decision = [
            'client_id' => $clientId,
            'request_type' => $requestType,
            'amount' => $amount,
            'credit_score' => $score['score'],
            'grade' => $score['grade'],
            'recommended' => $this->isRecommended($score, $requestType, $amount),
            'auto_decision' => null,
            'requires_review' => false,
            'decision_reason' => '',
            'made_at' => date('Y-m-d H:i:s')
        ];
        
        // Automatic decisions based on rules
        if ($score['score'] >= 850 && $amount <= 1000) {
            $decision['auto_decision'] = 'approved';
            $decision['decision_reason'] = 'Excellent score, within limits';
        } elseif ($score['score'] >= 700 && $amount <= 500) {
            $decision['auto_decision'] = 'approved';
            $decision['decision_reason'] = 'Good score, small amount';
        } elseif ($score['score'] < 500) {
            $decision['auto_decision'] = 'declined';
            $decision['decision_reason'] = 'Score too low for automatic approval';
            $decision['requires_review'] = true;
        } elseif ($this->hasRecentDecline($clientId)) {
            $decision['requires_review'] = true;
            $decision['decision_reason'] = 'Recent decline requires manual review';
        } else {
            $decision['requires_review'] = true;
            $decision['decision_reason'] = 'Manual review required';
        }
        
        $this->logDecision($decision);
        return $decision;
    }
    
    private function isRecommended($score, $requestType, $amount) {
        $recommendations = [
            'excellent' => ['max_amount' => 10000, 'types' => ['upgrade', 'addon', 'prepay']],
            'very_good' => ['max_amount' => 5000, 'types' => ['upgrade', 'addon']],
            'good' => ['max_amount' => 2500, 'types' => ['addon']],
            'fair' => ['max_amount' => 1000, 'types' => ['addon']],
            'poor' => ['max_amount' => 500, 'types' => ['addon']],
            'very_poor' => ['max_amount' => 0, 'types' => []]
        ];
        
        $rec = $recommendations[$score['grade']] ?? ['max_amount' => 0, 'types' => []];
        
        if ($amount > $rec['max_amount']) {
            return false;
        }
        
        if (!in_array($requestType, $rec['types'])) {
            return false;
        }
        
        return true;
    }
    
    private function hasRecentDecline($clientId) {
        $result = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_credit_decisions
             WHERE client_id = ? AND decision = 'declined'
             AND created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)",
            [$clientId]
        );
        
        return $result['count'] > 0;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_credit_scores (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    score INT NOT NULL,
    grade VARCHAR(20) NOT NULL,
    factors JSON,
    recommendations JSON,
    calculated_at DATETIME,
    INDEX idx_client (client_id),
    INDEX idx_score (score)
);

CREATE TABLE mod_credit_limits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL UNIQUE,
    limit_amount DECIMAL(10,2) NOT NULL DEFAULT 0,
    current_balance DECIMAL(10,2) DEFAULT 0,
    reserved_amount DECIMAL(10,2) DEFAULT 0,
    updated_at DATETIME,
    updated_by INT,
    INDEX idx_client (client_id)
);

CREATE TABLE mod_credit_reservations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    description VARCHAR(255),
    reserved_at DATETIME,
    released_at DATETIME,
    status ENUM('active', 'released', 'converted') DEFAULT 'active',
    INDEX idx_client (client_id)
);

CREATE TABLE mod_credit_decisions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    request_type VARCHAR(50),
    amount DECIMAL(10,2),
    credit_score INT,
    grade VARCHAR(20),
    auto_decision VARCHAR(20),
    requires_review TINYINT(1) DEFAULT 0,
    decision VARCHAR(20),
    decision_reason TEXT,
    reviewed_by INT,
    reviewed_at DATETIME,
    created_at DATETIME,
    INDEX idx_client (client_id),
    INDEX idx_created (created_at)
);

CREATE TABLE mod_credit_modifiers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    modifier_type VARCHAR(50),
    multiplier DECIMAL(5,2) DEFAULT 1.00,
    adjustment DECIMAL(10,2) DEFAULT 0,
    reason VARCHAR(255),
    active TINYINT(1) DEFAULT 1,
    created_at DATETIME,
    expires_at DATETIME,
    INDEX idx_client (client_id)
);

CREATE TABLE mod_credit_history (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    action_type VARCHAR(50),
    old_value DECIMAL(10,2),
    new_value DECIMAL(10,2),
    reason TEXT,
    performed_by INT,
    performed_at DATETIME,
    INDEX idx_client (client_id)
);
```

## Usage Examples

### Calculate Client Credit Score
```php
$scorer = new CreditScoringSystem();
$score = $scorer->calculateScore($clientId);

echo "Credit Score: {$score['score']}\n";
echo "Grade: {$score['grade']}\n";

foreach ($score['recommendations'] as $rec) {
    echo "- Improve {$rec['factor']}: {$rec['suggestion']}\n";
}
```

### Check Credit for Order
```php
$limitManager = new CreditLimitManager();
$check = $limitManager->checkCreditAvailability($clientId, 500);

if (!$check['available']) {
    echo "Insufficient credit. Available: {$check['available_credit']}";
}
```

### Make Credit Decision
```php
$engine = new CreditDecisionEngine();
$decision = $engine->makeDecision($clientId, 'upgrade', 200);

if ($decision['auto_decision'] === 'approved') {
    // Process automatically
} elseif ($decision['requires_review']) {
    // Queue for manual review
}
```

## Best Practices

1. **Recalculate regularly**: Update scores monthly or after significant events
2. **Use multiple factors**: Don't rely on a single metric
3. **Set clear thresholds**: Document approval/decline criteria
4. **Log all decisions**: Maintain audit trail for compliance
5. **Provide feedback**: Show clients how to improve scores
6. **Review automation**: Regularly audit automatic decision accuracy
7. **Monitor patterns**: Track declining patterns and fraud indicators