# WHMCS Compliance Monitoring Skill

## Purpose
Provides patterns for implementing compliance monitoring in WHMCS, tracking regulatory requirements, managing compliance status, and automating compliance reporting.

## Implementation Patterns

### Compliance Framework
```php
<?php
// includes/ComplianceMonitoring.class.php

class ComplianceMonitor {
    private $db;
    private $frameworks;
    
    public function __construct() {
        $this->db = console::db();
        $this->initializeFrameworks();
    }
    
    private function initializeFrameworks() {
        $this->frameworks = [
            'gdpr' => [
                'name' => 'General Data Protection Regulation',
                'jurisdiction' => 'EU',
                'requirements' => [
                    'data_consent', 'data_portability', 'right_to_erasure',
                    'privacy_policy', 'data_breach_notification', 'dpo_appointment'
                ]
            ],
            'pci_dss' => [
                'name' => 'Payment Card Industry Data Security Standard',
                'jurisdiction' => 'Global',
                'requirements' => [
                    'secure_network', 'cardholder_data_protection', 
                    'vulnerability_management', 'access_control', 'monitoring_testing'
                ]
            ],
            'soc2' => [
                'name' => 'SOC 2 Type II',
                'jurisdiction' => 'Global',
                'requirements' => [
                    'security', 'availability', 'confidentiality', 
                    'processing_integrity', 'privacy'
                ]
            ],
            'hipaa' => [
                'name' => 'Health Insurance Portability Act',
                'jurisdiction' => 'US',
                'requirements' => [
                    'administrative_safeguards', 'physical_safeguards', 
                    'technical_safeguards', 'policies_procedures'
                ]
            ]
        ];
    }
    
    // Register compliance requirement
    public function registerRequirement($framework, $requirement, $config = []) {
        $requirementId = $this->db->insert('mod_compliance_requirements', [
            'framework' => $framework,
            'requirement' => $requirement,
            'description' => $config['description'] ?? null,
            'frequency' => $config['frequency'] ?? 'quarterly',
            'evidence_required' => $config['evidence_required'] ?? true,
            'auto_verifiable' => $config['auto_verifiable'] ?? false,
            'risk_level' => $config['risk_level'] ?? 'medium',
            'created_at' => date('Y-m-d H:i:s'),
            'is_active' => 1
        ]);
        
        // Create initial compliance check schedule
        $this->scheduleComplianceCheck($requirementId);
        
        return $requirementId;
    }
    
    // Track compliance status for a client
    public function trackClientCompliance($clientId, $framework) {
        $requirements = $this->getFrameworkRequirements($framework);
        $status = [];
        
        foreach ($requirements as $req) {
            $status[$req] = $this->checkRequirementStatus($clientId, $req);
        }
        
        $overall = $this->calculateComplianceScore($status);
        
        $tracking = [
            'client_id' => $clientId,
            'framework' => $framework,
            'requirements' => json_encode($status),
            'compliance_score' => $overall,
            'last_checked' => date('Y-m-d H:i:s')
        ];
        
        $this->saveComplianceTracking($clientId, $framework, $tracking);
        
        return $tracking;
    }
    
    private function checkRequirementStatus($clientId, $requirement) {
        $method = "check_{$requirement}";
        
        if (method_exists($this, $method)) {
            $result = $this->$method($clientId);
        } else {
            $result = $this->genericRequirementCheck($clientId, $requirement);
        }
        
        $this->logCheckResult($clientId, $requirement, $result);
        
        return $result;
    }
    
    private function check_data_consent($clientId) {
        $consent = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_data_consents
             WHERE client_id = ? AND purpose IN ('marketing', 'analytics', 'third_party')
             AND consent_given = 1 AND (expires_at IS NULL OR expires_at > NOW())",
            [$clientId]
        );
        
        return [
            'compliant' => $consent['count'] >= 2, // Marketing + one other
            'consent_count' => $consent['count'],
            'checked_at' => date('Y-m-d H:i:s')
        ];
    }
    
    private function check_data_portability($clientId) {
        $exports = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_data_exports
             WHERE client_id = ? AND created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)",
            [$clientId]
        );
        
        $lastExport = $this->db->select(
            "SELECT created_at FROM mod_data_exports
             WHERE client_id = ? ORDER BY created_at DESC LIMIT 1",
            [$clientId]
        );
        
        return [
            'compliant' => true, // System supports export
            'export_available' => true,
            'last_export' => $lastExport['created_at'] ?? null,
            'checked_at' => date('Y-m-d H:i:s')
        ];
    }
    
    private function check_right_to_erasure($clientId) {
        $pendingRequests = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_erasure_requests
             WHERE client_id = ? AND status IN ('pending', 'in_progress')",
            [$clientId]
        );
        
        $completedRequests = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_erasure_requests
             WHERE client_id = ? AND status = 'completed'
             AND completed_at > DATE_SUB(NOW(), INTERVAL 30 DAY)",
            [$clientId]
        );
        
        return [
            'compliant' => $pendingRequests['count'] == 0,
            'pending_requests' => $pendingRequests['count'],
            'recent_completions' => $completedRequests['count'],
            'checked_at' => date('Y-m-d H:i:s')
        ];
    }
    
    private function check_privacy_policy($clientId) {
        $policyVersion = $this->db->select(
            "SELECT version, effective_date FROM mod_privacy_policies
             WHERE status = 'active' ORDER BY effective_date DESC LIMIT 1"
        );
        
        $clientAcceptance = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_policy_acceptances
             WHERE client_id = ? AND policy_version = ?",
            [$clientId, $policyVersion['version'] ?? null]
        );
        
        $isRecent = strtotime($policyVersion['effective_date']) > strtotime('-365 days');
        
        return [
            'compliant' => $clientAcceptance['count'] > 0 && $isRecent,
            'policy_version' => $policyVersion['version'] ?? null,
            'accepted' => $clientAcceptance['count'] > 0,
            'checked_at' => date('Y-m-d H:i:s')
        ];
    }
    
    private function check_data_breach_notification($clientId) {
        $breaches = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_data_breaches
             WHERE client_id = ? AND reported_at > DATE_SUB(NOW(), INTERVAL 12 MONTH)",
            [$clientId]
        );
        
        if ($breaches['count'] > 0) {
            $lastBreach = $this->db->select(
                "SELECT notification_sent, notification_time FROM mod_data_breaches
                 WHERE client_id = ? ORDER BY reported_at DESC LIMIT 1",
                [$clientId]
            );
            
            return [
                'compliant' => $lastBreach['notification_sent'] == 1,
                'breach_count' => $breaches['count'],
                'notification_sent' => $lastBreach['notification_sent'] ?? false,
                'checked_at' => date('Y-m-d H:i:s')
            ];
        }
        
        return [
            'compliant' => true,
            'breach_count' => 0,
            'checked_at' => date('Y-m-d H:i:s')
        ];
    }
    
    private function calculateComplianceScore($requirements) {
        $compliant = 0;
        $total = count($requirements);
        
        foreach ($requirements as $status) {
            if ($status['compliant']) {
                $compliant++;
            }
        }
        
        return $total > 0 ? ($compliant / $total) * 100 : 0;
    }
}
```

### Compliance Reporting
```php
class ComplianceReporter {
    public function generateReport($framework, $period = 'quarterly', $format = 'detailed') {
        $data = $this->gatherReportData($framework, $period);
        
        $report = [
            'framework' => $framework,
            'period' => $period,
            'generated_at' => date('Y-m-d H:i:s'),
            'summary' => $this->generateSummary($data),
            'requirements' => $data['requirements'],
            'clients' => $data['clients'],
            'evidence' => $data['evidence'],
            'gaps' => $this->identifyGaps($data),
            'recommendations' => $this->generateRecommendations($data)
        ];
        
        if ($format === 'summary') {
            return $this->formatSummaryReport($report);
        }
        
        return $report;
    }
    
    private function gatherReportData($framework, $period) {
        $startDate = $this->getPeriodStart($period);
        
        return [
            'requirements' => $this->getRequirementsStatus($framework),
            'clients' => $this->getClientCompliance($framework, $startDate),
            'evidence' => $this->getEvidenceSummary($framework, $startDate),
            'exceptions' => $this->getExceptions($framework, $startDate),
            'audits' => $this->getAuditResults($framework, $startDate)
        ];
    }
    
    private function generateSummary($data) {
        $totalClients = count($data['clients']);
        $compliantClients = count(array_filter($data['clients'], fn($c) => $c['score'] >= 80));
        
        $totalRequirements = count($data['requirements']);
        $metRequirements = count(array_filter($data['requirements'], fn($r) => $r['status'] === 'met'));
        
        return [
            'overall_compliance' => $totalClients > 0 
                ? ($compliantClients / $totalClients) * 100 
                : 0,
            'total_clients' => $totalClients,
            'compliant_clients' => $compliantClients,
            'total_requirements' => $totalRequirements,
            'met_requirements' => $metRequirements,
            'pending_evidence' => count($data['evidence']['pending']),
            'open_exceptions' => count($data['exceptions']),
            'audit_findings' => count($data['audits']['findings'])
        ];
    }
    
    public function exportEvidence($requirementId, $format = 'pdf') {
        $evidence = $this->getRequirementEvidence($requirementId);
        
        switch ($format) {
            case 'pdf':
                return $this->generatePDFReport($evidence);
            case 'csv':
                return $this->generateCSVExport($evidence);
            case 'json':
                return json_encode($evidence, JSON_PRETTY_PRINT);
            default:
                return $evidence;
        }
    }
}
```

### Compliance Alert System
```php
class ComplianceAlertSystem {
    public function checkComplianceStatus() {
        $alerts = [];
        
        // Check for expiring consents
        $expiringConsents = $this->getExpiringConsents();
        foreach ($expiringConsents as $consent) {
            $alerts[] = $this->createAlert('consent_expiring', $consent);
        }
        
        // Check for overdue compliance checks
        $overdueChecks = $this->getOverdueComplianceChecks();
        foreach ($overdueChecks as $check) {
            $alerts[] = $this->createAlert('check_overdue', $check);
        }
        
        // Check for expiring certifications
        $expiringCerts = $this->getExpiringCertifications();
        foreach ($expiringCerts as $cert) {
            $alerts[] = $this->createAlert('certification_expiring', $cert);
        }
        
        // Check for compliance gaps
        $gaps = $this->getComplianceGaps();
        foreach ($gaps as $gap) {
            $alerts[] = $this->createAlert('compliance_gap', $gap);
        }
        
        // Save and notify
        foreach ($alerts as $alert) {
            $this->saveAlert($alert);
            if ($alert['severity'] === 'critical') {
                $this->notifyComplianceTeam($alert);
            }
        }
        
        return $alerts;
    }
    
    private function createAlert($type, $data) {
        $templates = [
            'consent_expiring' => [
                'severity' => 'high',
                'message' => 'Client consent expiring soon',
                'template' => '{client_name}: {consent_type} consent expires on {date}'
            ],
            'check_overdue' => [
                'severity' => 'high',
                'message' => 'Compliance check overdue',
                'template' => '{requirement} check for {framework} is {days} days overdue'
            ],
            'certification_expiring' => [
                'severity' => 'critical',
                'message' => 'Security certification expiring',
                'template' => '{certification} expires in {days} days'
            ],
            'compliance_gap' => [
                'severity' => 'medium',
                'message' => 'Compliance gap detected',
                'template' => '{requirement} not met for {count} clients'
            ]
        ];
        
        $template = $templates[$type];
        
        return [
            'type' => $type,
            'severity' => $template['severity'],
            'message' => $template['message'],
            'data' => $data,
            'created_at' => date('Y-m-d H:i:s')
        ];
    }
    
    private function notifyComplianceTeam($alert) {
        // Email notification
        $this->sendEmailNotification($alert);
        
        // Dashboard notification
        $this->createDashboardNotification($alert);
        
        // Slack/webhook notification if configured
        $this->sendWebhookNotification($alert);
    }
}
```

### Compliance Automation
```php
class ComplianceAutomation {
    // Automate periodic compliance checks
    public function runScheduledChecks() {
        $dueChecks = $this->getDueChecks();
        
        foreach ($dueChecks as $check) {
            $this->executeComplianceCheck($check);
        }
    }
    
    private function executeComplianceCheck($check) {
        $monitor = new ComplianceMonitor();
        
        if ($check['auto_verifiable']) {
            $result = $monitor->checkRequirementStatus(
                $check['client_id'] ?? null, 
                $check['requirement']
            );
        } else {
            $result = $this->createManualReviewTask($check);
        }
        
        $this->updateCheckStatus($check['id'], $result);
        
        if (!$result['compliant'] && $check['risk_level'] === 'high') {
            $this->triggerRemediation($check, $result);
        }
        
        return $result;
    }
    
    // Automate consent management
    public function manageConsentCycle() {
        $this->sendConsentRequests();
        $this->processExpiredConsents();
        $this->updateConsentStatus();
    }
    
    private function sendConsentRequests() {
        $pendingRequests = $this->db->select(
            "SELECT c.id, c.email, c.firstname, c.lastname
             FROM tblclients c
             WHERE c.id NOT IN (
                 SELECT client_id FROM mod_data_consents 
                 WHERE purpose = 'marketing' AND consent_given = 1
             )
             AND c.email_opt_out = 0"
        );
        
        foreach ($pendingRequests as $client) {
            $this->queueConsentEmail($client);
        }
    }
    
    private function processExpiredConsents() {
        $expired = $this->db->select(
            "SELECT * FROM mod_data_consents
             WHERE expires_at < NOW() AND consent_given = 1"
        );
        
        foreach ($expired as $consent) {
            $this->db->where('id', $consent['id'])
                ->update('mod_data_consents', [
                    'consent_given' => 0,
                    'withdrawn_at' => date('Y-m-d H:i:s')
                ]);
            
            logActivity("Consent expired for client {$consent['client_id']}: {$consent['purpose']}");
        }
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_compliance_frameworks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    jurisdiction VARCHAR(100),
    description TEXT,
    requirements JSON,
    created_at DATETIME,
    is_active TINYINT(1) DEFAULT 1
);

CREATE TABLE mod_compliance_requirements (
    id INT AUTO_INCREMENT PRIMARY KEY,
    framework VARCHAR(50) NOT NULL,
    requirement VARCHAR(100) NOT NULL,
    description TEXT,
    frequency ENUM('weekly', 'monthly', 'quarterly', 'annually') DEFAULT 'quarterly',
    evidence_required TINYINT(1) DEFAULT 1,
    auto_verifiable TINYINT(1) DEFAULT 0,
    risk_level ENUM('low', 'medium', 'high', 'critical') DEFAULT 'medium',
    created_at DATETIME,
    is_active TINYINT(1) DEFAULT 1,
    INDEX idx_framework (framework)
);

CREATE TABLE mod_compliance_tracking (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT,
    framework VARCHAR(50) NOT NULL,
    requirements_status JSON,
    compliance_score DECIMAL(5,2),
    last_checked DATETIME,
    next_check DATE,
    UNIQUE KEY idx_client_framework (client_id, framework),
    INDEX idx_score (compliance_score)
);

CREATE TABLE mod_compliance_evidence (
    id INT AUTO_INCREMENT PRIMARY KEY,
    requirement_id INT NOT NULL,
    client_id INT,
    evidence_type VARCHAR(50),
    evidence_data TEXT,
    uploaded_by INT,
    uploaded_at DATETIME,
    verified TINYINT(1) DEFAULT 0,
    verified_by INT,
    verified_at DATETIME,
    INDEX idx_requirement (requirement_id),
    INDEX idx_client (client_id)
);

CREATE TABLE mod_compliance_alerts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    alert_type VARCHAR(50) NOT NULL,
    severity ENUM('low', 'medium', 'high', 'critical') NOT NULL,
    client_id INT,
    requirement_id INT,
    message TEXT,
    data JSON,
    created_at DATETIME,
    acknowledged_at DATETIME,
    acknowledged_by INT,
    resolved_at DATETIME,
    INDEX idx_type_severity (alert_type, severity),
    INDEX idx_unacknowledged (acknowledged_at)
);

CREATE TABLE mod_data_consents (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    purpose VARCHAR(50) NOT NULL,
    consent_given TINYINT(1) DEFAULT 0,
    consent_method VARCHAR(50),
    given_at DATETIME,
    expires_at DATETIME,
    withdrawn_at DATETIME,
    ip_address VARCHAR(45),
    user_agent VARCHAR(255),
    INDEX idx_client_purpose (client_id, purpose)
);

CREATE TABLE mod_erasure_requests (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    request_type VARCHAR(50),
    requested_at DATETIME,
    status ENUM('pending', 'in_progress', 'completed', 'cancelled') DEFAULT 'pending',
    completed_at DATETIME,
    verification_method VARCHAR(50),
    INDEX idx_status (status)
);

CREATE TABLE mod_policy_acceptances (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    policy_version VARCHAR(50) NOT NULL,
    accepted_at DATETIME,
    ip_address VARCHAR(45),
    INDEX idx_client (client_id)
);
```

## Usage Examples

### Track Client GDPR Compliance
```php
$monitor = new ComplianceMonitor();
$status = $monitor->trackClientCompliance($clientId, 'gdpr');

echo "GDPR Compliance Score: {$status['compliance_score']}%\n";
foreach ($status['requirements'] as $req => $check) {
    echo "  {$req}: " . ($check['compliant'] ? 'Compliant' : 'Non-compliant') . "\n";
}
```

### Generate Compliance Report
```php
$reporter = new ComplianceReporter();
$report = $reporter->generateReport('gdpr', 'quarterly', 'detailed');

echo "Overall Compliance: {$report['summary']['overall_compliance']}%\n";
echo "Compliant Clients: {$report['summary']['compliant_clients']}/{$report['summary']['total_clients']}\n";
```

### Run Scheduled Compliance Checks
```php
$automation = new ComplianceAutomation();
$automation->runScheduledChecks();
$automation->manageConsentCycle();
```

## Integration Points

- WHMCS client management for consent tracking
- Email system for consent requests and notifications
- Document management for evidence storage
- Audit logging for compliance records
- Dashboard widgets for compliance status
- Webhook integrations for external notifications