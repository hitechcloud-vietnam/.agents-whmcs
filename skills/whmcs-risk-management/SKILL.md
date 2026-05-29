# WHMCS Risk Management Skill

## Purpose
Provides patterns for implementing comprehensive risk management in WHMCS, including risk identification, assessment, mitigation planning, and continuous monitoring.

## Implementation Patterns

### Risk Assessment Engine
```php
<?php
// includes/RiskManagement.class.php

class RiskAssessmentEngine {
    private $db;
    private $riskCategories;
    
    public function __construct() {
        $this->db = console::db();
        $this->initializeCategories();
    }
    
    private function initializeCategories() {
        $this->riskCategories = [
            'financial' => ['weight' => 0.30, 'indicators' => ['payment_failure', 'high_balance', 'disputes']],
            'operational' => ['weight' => 0.25, 'indicators' => ['service_outage', 'support_overload', 'compliance_issue']],
            'security' => ['weight' => 0.25, 'indicators' => ['suspicious_activity', 'unauthorized_access', 'data_breach']],
            'reputational' => ['weight' => 0.20, 'indicators' => ['negative_feedback', 'public_complaints', 'social_media']]
        ];
    }
    
    // Perform comprehensive risk assessment
    public function assessClientRisk($clientId) {
        $assessment = [
            'client_id' => $clientId,
            'overall_risk_score' => 0,
            'risk_level' => 'low',
            'category_scores' => [],
            'risk_factors' => [],
            'recommendations' => [],
            'next_review' => null,
            'assessed_at' => date('Y-m-d H:i:s')
        ];
        
        foreach ($this->riskCategories as $category => $config) {
            $score = $this->assessCategoryRisk($clientId, $category);
            $assessment['category_scores'][$category] = $score;
            $assessment['overall_risk_score'] += $score['score'] * $config['weight'];
        }
        
        $assessment['risk_level'] = $this->determineRiskLevel($assessment['overall_risk_score']);
        $assessment['risk_factors'] = $this->identifyRiskFactors($clientId);
        $assessment['recommendations'] = $this->generateMitigationRecommendations($assessment);
        $assessment['next_review'] = $this->calculateNextReviewDate($assessment['risk_level']);
        
        $this->saveAssessment($assessment);
        
        return $assessment;
    }
    
    private function assessCategoryRisk($clientId, $category) {
        $indicators = $this->riskCategories[$category]['indicators'];
        $scores = [];
        
        foreach ($indicators as $indicator) {
            $scores[$indicator] = $this->evaluateIndicator($clientId, $indicator, $category);
        }
        
        $avgScore = array_sum($scores) / count($scores);
        $severity = $this->determineSeverity($avgScore);
        
        return [
            'score' => $avgScore,
            'severity' => $severity,
            'indicators' => $scores
        ];
    }
    
    private function evaluateIndicator($clientId, $indicator, $category) {
        $method = "evaluate_{$indicator}";
        if (method_exists($this, $method)) {
            return $this->$method($clientId);
        }
        
        // Fallback to generic evaluation
        return $this->genericIndicatorEvaluation($clientId, $indicator, $category);
    }
    
    private function evaluate_payment_failure($clientId) {
        $result = $this->db->select(
            "SELECT 
                COUNT(*) as total,
                SUM(CASE WHEN status = 'Failed' THEN 1 ELSE 0 END) as failed,
                SUM(CASE WHEN status = 'Failed' AND attempts >= 3 THEN 1 ELSE 0 END) as multiple_failed
             FROM mod_payment_attempts
             WHERE client_id = ? AND created_at > DATE_SUB(NOW(), INTERVAL 90 DAY)",
            [$clientId]
        );
        
        if ($result['total'] == 0) return 0;
        
        $failureRate = $result['failed'] / $result['total'];
        $multipleFailedPenalty = $result['multiple_failed'] * 0.2;
        
        return min(100, $failureRate * 50 + $multipleFailedPenalty * 50);
    }
    
    private function evaluate_high_balance($clientId) {
        $limit = $this->getCreditLimit($clientId);
        $balance = $this->getOutstandingBalance($clientId);
        
        if ($limit <= 0) return 50; // No limit set
        
        $utilization = $balance / $limit;
        
        if ($utilization > 1.0) return 100;
        if ($utilization > 0.8) return 80;
        if ($utilization > 0.5) return 50;
        return 0;
    }
    
    private function evaluate_disputes($clientId) {
        $result = $this->db->select(
            "SELECT COUNT(*) as count FROM tbltickets
             WHERE userid = ? AND (subject LIKE '%dispute%' OR subject LIKE '%chargeback%')
             AND created_at > DATE_SUB(NOW(), INTERVAL 180 DAY)",
            [$clientId]
        );
        
        $count = $result['count'];
        
        if ($count >= 5) return 100;
        if ($count >= 3) return 70;
        if ($count >= 1) return 40;
        return 0;
    }
    
    private function evaluate_service_outage($clientId) {
        $result = $this->db->select(
            "SELECT SUM(duration_minutes) as total_downtime
             FROM mod_service_events
             WHERE client_id = ? AND event_type = 'downtime'
             AND start_time > DATE_SUB(NOW(), INTERVAL 30 DAY)",
            [$clientId]
        );
        
        $downtime = $result['total_downtime'] ?? 0;
        
        if ($downtime >= 1440) return 100; // 24+ hours
        if ($downtime >= 720) return 70;   // 12+ hours
        if ($downtime >= 360) return 40;   // 6+ hours
        return 0;
    }
    
    private function evaluate_suspicious_activity($clientId) {
        $result = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_security_events
             WHERE client_id = ? AND severity IN ('medium', 'high', 'critical')
             AND created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)",
            [$clientId]
        );
        
        $count = $result['count'];
        
        if ($count >= 5) return 100;
        if ($count >= 3) return 70;
        if ($count >= 1) return 40;
        return 0;
    }
    
    private function determineSeverity($score) {
        if ($score >= 80) return 'critical';
        if ($score >= 60) return 'high';
        if ($score >= 40) return 'medium';
        if ($score >= 20) return 'low';
        return 'minimal';
    }
    
    private function determineRiskLevel($score) {
        if ($score >= 75) return 'critical';
        if ($score >= 60) return 'high';
        if ($score >= 40) return 'medium';
        if ($score >= 20) return 'low';
        return 'minimal';
    }
    
    private function identifyRiskFactors($clientId) {
        $factors = [];
        
        // Check payment history
        $paymentIssues = $this->getPaymentIssues($clientId);
        if (count($paymentIssues) > 0) {
            $factors[] = [
                'category' => 'financial',
                'type' => 'payment_pattern',
                'details' => $paymentIssues,
                'severity' => count($paymentIssues) > 3 ? 'high' : 'medium'
            ];
        }
        
        // Check account behavior
        $behavior = $this->analyzeAccountBehavior($clientId);
        if ($behavior['score'] > 60) {
            $factors[] = [
                'category' => 'security',
                'type' => 'behavior_anomaly',
                'details' => $behavior,
                'severity' => 'high'
            ];
        }
        
        // Check support patterns
        $supportPatterns = $this->analyzeSupportPatterns($clientId);
        if ($supportPatterns['high_volume']) {
            $factors[] = [
                'category' => 'operational',
                'type' => 'support_overload',
                'details' => $supportPatterns,
                'severity' => 'medium'
            ];
        }
        
        return $factors;
    }
    
    private function generateMitigationRecommendations($assessment) {
        $recommendations = [];
        
        // Financial risk recommendations
        if ($assessment['category_scores']['financial']['score'] > 50) {
            $recommendations[] = [
                'category' => 'financial',
                'action' => 'Review credit limit',
                'priority' => 'high',
                'details' => 'Client showing elevated financial risk indicators'
            ];
            
            $recommendations[] = [
                'category' => 'financial',
                'action' => 'Require upfront payment',
                'priority' => 'medium',
                'details' => 'Consider requiring advance payments for new orders'
            ];
        }
        
        // Security risk recommendations
        if ($assessment['category_scores']['security']['score'] > 50) {
            $recommendations[] = [
                'category' => 'security',
                'action' => 'Enable additional verification',
                'priority' => 'high',
                'details' => 'Implement additional authentication measures'
            ];
            
            $recommendations[] = [
                'category' => 'security',
                'action' => 'Monitor activity closely',
                'priority' => 'high',
                'details' => 'Increase monitoring frequency and set up alerts'
            ];
        }
        
        return $recommendations;
    }
}
```

### Risk Monitoring System
```php
class RiskMonitoringSystem {
    private $db;
    
    // Monitor for new risk events
    public function monitorRiskEvents() {
        $events = $this->getNewRiskEvents();
        
        foreach ($events as $event) {
            $this->assessEventImpact($event);
            
            if ($event['risk_score'] >= 60) {
                $this->triggerAlert($event);
            }
            
            if ($event['risk_score'] >= 80) {
                $this->executeAutomatedResponse($event);
            }
        }
    }
    
    private function getNewRiskEvents() {
        return $this->db->select(
            "SELECT * FROM mod_risk_events
             WHERE processed = 0 AND created_at > DATE_SUB(NOW(), INTERVAL 1 DAY)
             ORDER BY created_at DESC"
        );
    }
    
    private function assessEventImpact($event) {
        $baseScore = $this->getEventBaseScore($event['event_type']);
        $contextMultiplier = $this->getContextMultiplier($event);
        $clientHistoryMultiplier = $this->getClientHistoryMultiplier($event['client_id']);
        
        $riskScore = min(100, $baseScore * $contextMultiplier * $clientHistoryMultiplier);
        
        $this->db->where('id', $event['id'])
            ->update('mod_risk_events', [
                'risk_score' => $riskScore,
                'processed' => 1,
                'processed_at' => date('Y-m-d H:i:s')
            ]);
        
        return $riskScore;
    }
    
    private function triggerAlert($event) {
        $alert = [
            'client_id' => $event['client_id'],
            'event_id' => $event['id'],
            'risk_score' => $event['risk_score'],
            'severity' => $this->determineSeverity($event['risk_score']),
            'message' => $this->generateAlertMessage($event),
            'created_at' => date('Y-m-d H:i:s'),
            'status' => 'pending'
        ];
        
        $this->db->insert('mod_risk_alerts', $alert);
        
        // Send notification
        $this->notifyRiskTeam($alert);
    }
    
    private function executeAutomatedResponse($event) {
        $responses = $this->getAutomatedResponses($event['event_type']);
        
        foreach ($responses as $response) {
            $this->executeResponse($response, $event);
        }
    }
    
    private function executeResponse($response, $event) {
        switch ($response['action']) {
            case 'suspend_services':
                $this->suspendHighRiskServices($event['client_id']);
                break;
            case 'require_verification':
                $this->requireIdentityVerification($event['client_id']);
                break;
            case 'limit_credit':
                $this->reduceCreditLimit($event['client_id'], 0.5);
                break;
            case 'flag_account':
                $this->flagAccountForReview($event['client_id']);
                break;
        }
        
        $this->logResponseExecution($response, $event);
    }
}
```

### Risk Reporting Dashboard
```php
class RiskReportingDashboard {
    public function getDashboardData($filters = []) {
        $defaults = [
            'period' => '30d',
            'risk_level' => null,
            'category' => null
        ];
        $config = array_merge($defaults, $filters);
        
        return [
            'overview' => $this->getOverviewMetrics($config),
            'risk_distribution' => $this->getRiskDistribution($config),
            'top_risks' => $this->getTopRisks($config),
            'trending_up' => $this->getTrendingRisks($config),
            'alerts_summary' => $this->getAlertsSummary($config),
            'mitigation_progress' => $this->getMitigationProgress($config)
        ];
    }
    
    private function getOverviewMetrics($config) {
        $startDate = $this->getStartDate($config['period']);
        
        $metrics = $this->db->select(
            "SELECT 
                COUNT(DISTINCT client_id) as total_clients,
                AVG(risk_score) as avg_risk_score,
                COUNT(CASE WHEN risk_level = 'critical' THEN 1 END) as critical_count,
                COUNT(CASE WHEN risk_level = 'high' THEN 1 END) as high_count,
                COUNT(CASE WHEN risk_level = 'medium' THEN 1 END) as medium_count,
                COUNT(CASE WHEN risk_level = 'low' THEN 1 END) as low_count
             FROM mod_risk_assessments
             WHERE assessed_at >= ?",
            [$startDate]
        );
        
        return $metrics;
    }
    
    private function getRiskDistribution($config) {
        return $this->db->select(
            "SELECT risk_level, COUNT(*) as count
             FROM mod_risk_assessments
             GROUP BY risk_level
             ORDER BY FIELD(risk_level, 'critical', 'high', 'medium', 'low', 'minimal')"
        );
    }
    
    private function getTopRisks($config) {
        return $this->db->select(
            "SELECT r.client_id, c.companyname, r.risk_level, r.risk_score,
                    r.category_scores, r.assessed_at
             FROM mod_risk_assessments r
             JOIN tblclients c ON c.id = r.client_id
             WHERE r.assessed_at >= ?
             ORDER BY r.risk_score DESC
             LIMIT 20",
            [$this->getStartDate($config['period'])]
        );
    }
    
    private function getTrendingRisks($config) {
        return $this->db->select(
            "SELECT category, 
                    AVG(score) as current_avg,
                    (SELECT AVG(score) FROM mod_risk_trends WHERE recorded_at >= DATE_SUB(NOW(), INTERVAL 60 DAY)) as previous_avg,
                    (AVG(score) - (SELECT AVG(score) FROM mod_risk_trends WHERE recorded_at >= DATE_SUB(NOW(), INTERVAL 60 DAY))) as trend
             FROM mod_risk_trends
             WHERE recorded_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
             GROUP BY category
             ORDER BY trend DESC"
        );
    }
    
    private function getMitigationProgress($config) {
        return $this->db->select(
            "SELECT 
                COUNT(*) as total_actions,
                COUNT(CASE WHEN status = 'completed' THEN 1 END) as completed,
                COUNT(CASE WHEN status = 'pending' THEN 1 END) as pending,
                COUNT(CASE WHEN status = 'overdue' THEN 1 END) as overdue,
                AVG(TIMESTAMPDIFF(HOUR, created_at, completed_at)) as avg_completion_hours
             FROM mod_risk_mitigation_actions
             WHERE created_at >= ?",
            [$this->getStartDate($config['period'])]
        );
    }
}
```

### Risk Mitigation Planner
```php
class RiskMitigationPlanner {
    public function createMitigationPlan($assessment) {
        $plan = [
            'assessment_id' => $assessment['id'],
            'client_id' => $assessment['client_id'],
            'actions' => [],
            'created_at' => date('Y-m-d H:i:s'),
            'target_completion' => null,
            'status' => 'draft'
        ];
        
        foreach ($assessment['recommendations'] as $rec) {
            $action = $this->createAction($rec, $assessment);
            $plan['actions'][] = $action;
        }
        
        $plan['target_completion'] = $this->calculateTargetDate($plan['actions']);
        
        $this->savePlan($plan);
        $this->assignResponsibilities($plan);
        
        return $plan;
    }
    
    private function createAction($recommendation, $assessment) {
        return [
            'category' => $recommendation['category'],
            'action' => $recommendation['action'],
            'details' => $recommendation['details'],
            'priority' => $recommendation['priority'],
            'assigned_to' => $this->determineAssignee($recommendation['category']),
            'due_date' => $this->calculateDueDate($recommendation['priority']),
            'status' => 'pending',
            'dependencies' => []
        ];
    }
    
    private function calculateDueDate($priority) {
        switch ($priority) {
            case 'critical':
                return date('Y-m-d', strtotime('+1 day'));
            case 'high':
                return date('Y-m-d', strtotime('+3 days'));
            case 'medium':
                return date('Y-m-d', strtotime('+7 days'));
            default:
                return date('Y-m-d', strtotime('+14 days'));
        }
    }
    
    public function trackMitigationProgress($planId) {
        $actions = $this->getPlanActions($planId);
        
        $progress = [
            'total' => count($actions),
            'completed' => count(array_filter($actions, fn($a) => $a['status'] === 'completed')),
            'pending' => count(array_filter($actions, fn($a) => $a['status'] === 'pending')),
            'overdue' => count(array_filter($actions, fn($a) => $a['status'] === 'overdue')),
            'completion_pct' => 0
        ];
        
        if ($progress['total'] > 0) {
            $progress['completion_pct'] = ($progress['completed'] / $progress['total']) * 100;
        }
        
        // Check for overdue actions
        foreach ($actions as &$action) {
            if ($action['status'] === 'pending' && $action['due_date'] < date('Y-m-d')) {
                $action['status'] = 'overdue';
                $this->updateActionStatus($action['id'], 'overdue');
            }
        }
        
        return $progress;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_risk_assessments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    overall_risk_score DECIMAL(5,2) NOT NULL,
    risk_level ENUM('critical', 'high', 'medium', 'low', 'minimal') NOT NULL,
    category_scores JSON,
    risk_factors JSON,
    recommendations JSON,
    next_review DATE,
    assessed_at DATETIME,
    UNIQUE KEY idx_client (client_id),
    INDEX idx_risk_level (risk_level),
    INDEX idx_assessed (assessed_at)
);

CREATE TABLE mod_risk_events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_data JSON,
    risk_score DECIMAL(5,2),
    processed TINYINT(1) DEFAULT 0,
    created_at DATETIME,
    processed_at DATETIME,
    INDEX idx_client (client_id),
    INDEX idx_unprocessed (processed, created_at)
);

CREATE TABLE mod_risk_alerts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    event_id INT,
    risk_score DECIMAL(5,2),
    severity ENUM('low', 'medium', 'high', 'critical') NOT NULL,
    message TEXT,
    created_at DATETIME,
    acknowledged_at DATETIME,
    acknowledged_by INT,
    INDEX idx_severity (severity),
    INDEX idx_unacknowledged (acknowledged_at)
);

CREATE TABLE mod_risk_mitigation_actions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    plan_id INT NOT NULL,
    category VARCHAR(50),
    action VARCHAR(255) NOT NULL,
    details TEXT,
    priority ENUM('low', 'medium', 'high', 'critical'),
    assigned_to INT,
    due_date DATE,
    status ENUM('pending', 'in_progress', 'completed', 'cancelled', 'overdue') DEFAULT 'pending',
    completed_at DATETIME,
    created_at DATETIME,
    INDEX idx_plan (plan_id),
    INDEX idx_status (status)
);

CREATE TABLE mod_risk_trends (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    category VARCHAR(50),
    score DECIMAL(5,2),
    recorded_at DATETIME,
    INDEX idx_category_time (category, recorded_at)
);
```

## Usage Examples

### Run Risk Assessment
```php
$engine = new RiskAssessmentEngine();
$assessment = $engine->assessClientRisk($clientId);

echo "Risk Level: {$assessment['risk_level']}\n";
echo "Risk Score: {$assessment['overall_risk_score']}\n";

foreach ($assessment['recommendations'] as $rec) {
    echo "[{$rec['priority']}] {$rec['action']}: {$rec['details']}\n";
}
```

### Monitor Risk Events
```php
$monitor = new RiskMonitoringSystem();
$monitor->monitorRiskEvents(); // Run via cron
```

### Generate Risk Report
```php
$dashboard = new RiskReportingDashboard();
$report = $dashboard->getDashboardData(['period' => '30d']);

echo "Critical Clients: {$report['overview']['critical_count']}\n";
echo "Average Risk Score: {$report['overview']['avg_risk_score']}\n";
```

## Best Practices

1. **Regular assessments**: Re-evaluate risk monthly or quarterly
2. **Multi-category analysis**: Consider financial, operational, security, and reputational risks
3. **Automated monitoring**: Set up real-time risk event detection
4. **Clear escalation paths**: Define actions for each risk level
5. **Document decisions**: Maintain audit trail for risk decisions
6. **Continuous improvement**: Update risk models based on outcomes
7. **Cross-functional collaboration**: Involve finance, ops, and security teams