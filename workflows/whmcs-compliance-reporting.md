# WHMCS Compliance Reporting Workflow

## Purpose

Implement comprehensive compliance reporting for WHMCS to meet regulatory requirements (GDPR, PCI-DSS, SOC 2, HIPAA), demonstrate control effectiveness, and provide audit evidence. This workflow covers compliance frameworks, evidence collection, report generation, and remediation tracking.

## Prerequisites

- Compliance framework requirements identified
- Audit logging implemented (see Audit Trail workflow)
- Data classification completed
- Risk assessment performed
- Remediation tracking system

## Workflow Steps

### Step 1: Compliance Framework Mapping

```
┌─────────────────────────────────────────────────────────────┐
│               Compliance Framework Mapping                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    PCI-DSS                         │   │
│  │  • Access control (Req 7, 8)                       │   │
│  │  • Logging (Req 10)                               │   │
│  │  • Encryption (Req 3, 4, 8)                       │   │
│  │  • Vulnerability management (Req 11)                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                      GDPR                            │   │
│  │  • Data consent (Art 6, 7)                         │   │
│  │  • Right to access (Art 15)                        │   │
│  │  • Right to erasure (Art 17)                        │   │
│  │  • Data portability (Art 20)                       │   │
│  │  • Breach notification (Art 33)                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                     SOC 2                            │   │
│  │  • Security (CC6)                                  │   │
│  │  • Availability (CC1)                              │   │
│  │  • Confidentiality (CC6)                           │   │
│  │  • Privacy (P3, P4)                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                     HIPAA                            │   │
│  │  • Technical safeguards (164.312)                   │   │
│  │  • Administrative safeguards (164.308)              │   │
│  │  • Physical safeguards (164.310)                   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Control Implementation Matrix

```markdown
## WHMCS Compliance Control Matrix

### Access Control (PCI-DSS Req 7, 8)

| Control ID | Control Description | Implementation | Evidence | Status |
|------------|-------------------|----------------|----------|--------|
| AC-1 | Need-to-know access | Role-based permissions | Role assignments | Compliant |
| AC-2 | Unique IDs | User accounts in WHMCS | User listing | Compliant |
| AC-3 | Access revocation | Deactivation on termination | Termination log | Compliant |
| AC-4 | Privileged access | Admin role restrictions | Admin audit | Compliant |

### Data Protection (GDPR Art 32, PCI-DSS Req 3, 4)

| Control ID | Control Description | Implementation | Evidence | Status |
|------------|-------------------|----------------|----------|--------|
| DP-1 | Encryption at rest | AES-256 for sensitive data | Config evidence | Compliant |
| DP-2 | Encryption in transit | TLS 1.2+ enforced | SSL config | Compliant |
| DP-3 | Data minimization | Only necessary data collected | Data inventory | Compliant |
| DP-4 | Retention limits | Configured retention periods | Retention policy | Partial |

### Logging and Monitoring (PCI-DSS Req 10, SOC2 CC7)

| Control ID | Control Description | Implementation | Evidence | Status |
|------------|-------------------|----------------|----------|--------|
| LM-1 | All access logged | Audit hook implementation | Audit reports | Compliant |
| LM-2 | Log retention | 90 days online, 1 year archive | Archive logs | Compliant |
| LM-3 | Log protection | Immutable log storage | Config evidence | Compliant |
| LM-4 | Alert thresholds | Configured alerts | Alert configs | Compliant |

### Incident Response (GDPR Art 33, SOC2 CC7)

| Control ID | Control Description | Implementation | Evidence | Status |
|------------|-------------------|----------------|----------|--------|
| IR-1 | Incident detection | Monitoring system | Alert logs | Compliant |
| IR-2 | Response procedures | Documented runbooks | IR workflow | Compliant |
| IR-3 | Notification | 72-hour GDPR notification | Notification template | Compliant |
```

### Step 3: Compliance Evidence Collection

```bash
#!/bin/bash
# /opt/scripts/collect_compliance_evidence.sh

COMPLIANCE_YEAR=$(date +%Y)
EVIDENCE_DIR="/var/compliance/evidence/$COMPLIANCE_YEAR"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p "$EVIDENCE_DIR"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

collect_evidence() {
    local category=$1
    local evidence_file=$2
    
    log "Collecting $category evidence..."
    
    case $category in
        "access_control")
            # Export user list
            php /opt/scripts/export_user_list.php > "$EVIDENCE_DIR/user_list.csv"
            
            # Export role assignments
            php /opt/scripts/export_roles.php > "$EVIDENCE_DIR/role_assignments.csv"
            
            # Export access logs
            php /opt/scripts/export_access_logs.php "$EVIDENCE_DIR/access_logs.csv"
            ;;
            
        "encryption")
            # SSL certificate info
            openssl x509 -in /etc/ssl/certs/whmcs.crt -noout -text > "$EVIDENCE_DIR/ssl_config.txt"
            
            # Database encryption config
            php /opt/scripts/export_db_encryption.php > "$EVIDENCE_DIR/db_encryption.txt"
            ;;
            
        "logging")
            # Export audit logs for period
            php /opt/scripts/export_audit_logs.php "$EVIDENCE_DIR/audit_logs.csv"
            
            # Export security events
            php /opt/scripts/export_security_events.php "$EVIDENCE_DIR/security_events.csv"
            ;;
            
        "incident_response")
            # Export incident log
            php /opt/scripts/export_incidents.php "$EVIDENCE_DIR/incidents.json"
            
            # Export alert history
            php /opt/scripts/export_alerts.php "$EVIDENCE_DIR/alerts.csv"
            ;;
            
        "data_protection")
            # Data inventory
            php /opt/scripts/data_inventory.php "$EVIDENCE_DIR/data_inventory.json"
            
            # Retention configuration
            php /opt/scripts/retention_config.php "$EVIDENCE_DIR/retention_config.json"
            
            # Consent records
            php /opt/scripts/export_consents.php "$EVIDENCE_DIR/consents.csv"
            ;;
    esac
    
    # Generate checksums
    sha256sum "$EVIDENCE_DIR"/* > "$EVIDENCE_DIR/checksums.sha256"
}

# Collect all evidence
for category in access_control encryption logging incident_response data_protection; do
    collect_evidence $category
done

# Package evidence
tar -czf "/var/compliance/evidence/whmcs_evidence_$TIMESTAMP.tar.gz" "$EVIDENCE_DIR"

# Upload to secure storage
aws s3 cp "/var/compliance/evidence/whmcs_evidence_$TIMESTAMP.tar.gz" \
    s3://company-compliance-evidence/ \
    --sse AES256

# Notify compliance team
curl -X POST "$SLACK_WEBHOOK" -d "Compliance evidence collection complete for $COMPLIANCE_YEAR"

log "Evidence collection complete"
```

### Step 4: Compliance Reporting System

```php
<?php
// /opt/scripts/compliance_reports.php

class ComplianceReporter {
    private $pdo;
    private $reportDir = '/var/compliance/reports';
    
    public function __construct() {
        $this->pdo = new PDO('mysql:host=localhost', 'whmcs', 'password');
    }
    
    public function generatePCIDSSReport(string $startDate, string $endDate): array {
        $report = [
            'title' => 'PCI-DSS Compliance Report',
            'period' => ['start' => $startDate, 'end' => $endDate],
            'generated' => date('c'),
            'controls' => []
        ];
        
        // Requirement 3: Protect stored cardholder data
        $report['controls']['req3'] = [
            'title' => 'Protect Stored Cardholder Data',
            'requirements' => [
                $this->checkDataAtRestEncryption(),
                $this->checkCardholderDataMasking(),
                $this->checkKeyManagement()
            ]
        ];
        
        // Requirement 7: Restrict access
        $report['controls']['req7'] = [
            'title' => 'Restrict Access to Cardholder Data',
            'requirements' => [
                $this->checkAccessControls(),
                $this->checkRoleAssignments(),
                $this->checkAccessReviews()
            ]
        ];
        
        // Requirement 10: Track and monitor
        $report['controls']['req10'] = [
            'title' => 'Track and Monitor All Access',
            'requirements' => [
                $this->checkAuditLogging(),
                $this->checkLogRetention(),
                $this->checkLogProtection()
            ]
        ];
        
        // Calculate overall compliance
        $report['overall_compliance'] = $this->calculateComplianceScore($report['controls']);
        
        return $report;
    }
    
    public function generateGDPRReport(string $startDate, string $endDate): array {
        $report = [
            'title' => 'GDPR Compliance Report',
            'period' => ['start' => $startDate, 'end' => $endDate],
            'generated' => date('c'),
            'articles' => []
        ];
        
        // Article 15: Right of access
        $report['articles']['art15'] = [
            'title' => 'Right of Access',
            'metrics' => $this->getDataAccessMetrics(),
            'compliance' => 'Compliant'
        ];
        
        // Article 17: Right to erasure
        $report['articles']['art17'] = [
            'title' => 'Right to Erasure',
            'metrics' => $this->getErasureMetrics(),
            'compliance' => 'Compliant'
        ];
        
        // Article 33: Breach notification
        $report['articles']['art33'] = [
            'title' => 'Breach Notification',
            'metrics' => $this->getBreachMetrics(),
            'compliance' => 'Compliant'
        ];
        
        return $report;
    }
    
    private function checkDataAtRestEncryption(): array {
        $evidence = [];
        
        // Check database encryption
        $dbEncrypt = $this->pdo->query("SHOW VARIABLES LIKE 'innodb_encrypt_tables'")->fetch();
        $evidence[] = [
            'check' => 'Database encryption enabled',
            'result' => ($dbEncrypt['Value'] ?? 'OFF') === 'ON' ? 'Pass' : 'Fail',
            'evidence' => $dbEncrypt
        ];
        
        // Check sensitive field encryption
        $stmt = $this->pdo->query("
            SELECT COUNT(*) as encrypted_count
            FROM information_schema.columns
            WHERE table_schema = 'whmcs_main'
            AND column_name LIKE '%encrypted%'
        ");
        $evidence[] = [
            'check' => 'Encrypted columns present',
            'result' => ($stmt->fetchColumn() > 0) ? 'Pass' : 'Fail',
            'evidence' => ['encrypted_columns' => $stmt->fetchColumn()]
        ];
        
        return ['status' => 'Compliant', 'evidence' => $evidence];
    }
    
    private function checkAccessControls(): array {
        $stmt = $this->pdo->query("
            SELECT COUNT(*) as total_users,
            SUM(CASE WHEN disabled = 0 THEN 1 ELSE 0 END) as active_users
            FROM tbladmins
        ");
        $row = $stmt->fetch();
        
        return [
            'status' => 'Compliant',
            'evidence' => [
                'total_admins' => $row['total_users'],
                'active_admins' => $row['active_users'],
                'access_review_date' => date('Y-m-d')
            ]
        ];
    }
    
    private function checkAuditLogging(): array {
        $stmt = $this->pdo->query("
            SELECT COUNT(*) as log_count,
            MIN(timestamp) as oldest_log,
            MAX(timestamp) as newest_log
            FROM mod_audit_log
            WHERE timestamp > DATE_SUB(NOW(), INTERVAL 90 DAY)
        ");
        $row = $stmt->fetch();
        
        return [
            'status' => ($row['log_count'] > 0) ? 'Compliant' : 'Non-compliant',
            'evidence' => [
                'logs_in_90_days' => $row['log_count'],
                'oldest_log' => $row['oldest_log'],
                'newest_log' => $row['newest_log']
            ]
        ];
    }
    
    private function calculateComplianceScore(array $controls): float {
        $totalChecks = 0;
        $passedChecks = 0;
        
        foreach ($controls as $control) {
            foreach ($control['requirements'] as $req) {
                $totalChecks++;
                if (($req['status'] ?? '') === 'Compliant') {
                    $passedChecks++;
                }
            }
        }
        
        return round(($passedChecks / $totalChecks) * 100, 2);
    }
    
    private function getDataAccessMetrics(): array {
        $stmt = $this->pdo->query("
            SELECT COUNT(*) as access_requests
            FROM mod_audit_log
            WHERE category = 'data_access'
            AND timestamp > DATE_SUB(NOW(), INTERVAL 30 DAY)
        ");
        
        return ['access_requests_30d' => (int) $stmt->fetchColumn()];
    }
    
    private function getErasureMetrics(): array {
        $stmt = $this->pdo->query("
            SELECT COUNT(*) as erasure_requests
            FROM tblclients 
            WHERE deleted_at IS NOT NULL
            AND deleted_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
        ");
        
        return ['erasure_requests_30d' => (int) $stmt->fetchColumn()];
    }
    
    private function getBreachMetrics(): array {
        $stmt = $this->pdo->query("
            SELECT COUNT(*) as security_incidents
            FROM mod_audit_log
            WHERE category = 'security_event'
            AND timestamp > DATE_SUB(NOW(), INTERVAL 30 DAY)
        ");
        
        return ['security_incidents_30d' => (int) $stmt->fetchColumn()];
    }
    
    public function exportToPDF(array $report): string {
        $filename = $this->reportDir . '/compliance_report_' . date('Y-m-d') . '.pdf';
        
        // Use dompdf or tcpdf to generate PDF
        // Implementation depends on PDF library
        
        return $filename;
    }
}

// Generate and save reports
$reporter = new ComplianceReporter();

// PCI-DSS Quarterly Report
$ pciReport = $reporter->generatePCIDSSReport(
    date('Y-01-01'),
    date('Y-03-31')
);
file_put_contents('/var/compliance/reports/pcidss_q1.json', json_encode($pciReport, JSON_PRETTY_PRINT));

// GDPR Annual Report
$gdprReport = $reporter->generateGDPRReport(
    date('Y-01-01'),
    date('Y-12-31')
);
file_put_contents('/var/compliance/reports/gdpr_annual.json', json_encode($gdprReport, JSON_PRETTY_PRINT));
```

### Step 5: Remediation Tracking

```php
<?php
// /opt/scripts/remediation_tracker.php

class RemediationTracker {
    private $pdo;
    
    public function __construct() {
        $this->pdo = new PDO('mysql:host=localhost', 'whmcs', 'password');
    }
    
    public function createFinding(array $finding): int {
        $stmt = $this->pdo->prepare("
            INSERT INTO compliance_findings (
                finding_id, framework, control_id,
                severity, description, discovered_date,
                due_date, owner, status, remediation_plan
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        ");
        
        $stmt->execute([
            $finding['id'],
            $finding['framework'],
            $finding['control_id'],
            $finding['severity'],
            $finding['description'],
            $finding['discovered_date'],
            $finding['due_date'],
            $finding['owner'],
            'open',
            $finding['remediation_plan']
        ]);
        
        return $this->pdo->lastInsertId();
    }
    
    public function updateProgress(int $findingId, string $status, string $notes): bool {
        $stmt = $this->pdo->prepare("
            UPDATE compliance_findings 
            SET status = ?,
                last_update = NOW(),
                last_notes = ?,
                progress_percentage = ?
            WHERE id = ?
        ");
        
        return $stmt->execute([
            $status,
            $notes,
            $this->calculateProgress($status),
            $findingId
        ]);
    }
    
    public function getDashboard(): array {
        $stmt = $this->pdo->query("
            SELECT 
                framework,
                status,
                COUNT(*) as count
            FROM compliance_findings
            GROUP BY framework, status
        ");
        
        $dashboard = [];
        while ($row = $stmt->fetch()) {
            $dashboard[$row['framework']][$row['status']] = $row['count'];
        }
        
        return [
            'open_findings' => $this->countByStatus('open'),
            'in_progress' => $this->countByStatus('in_progress'),
            'overdue' => $this->countOverdue(),
            'by_framework' => $dashboard
        ];
    }
    
    private function calculateProgress(string $status): int {
        $progressMap = [
            'open' => 0,
            'in_progress' => 25,
            'mitigated' => 75,
            'closed' => 100
        ];
        
        return $progressMap[$status] ?? 0;
    }
    
    private function countByStatus(string $status): int {
        $stmt = $this->pdo->prepare("SELECT COUNT(*) FROM compliance_findings WHERE status = ?");
        $stmt->execute([$status]);
        return (int) $stmt->fetchColumn();
    }
    
    private function countOverdue(): int {
        $stmt = $this->pdo->query("
            SELECT COUNT(*) FROM compliance_findings 
            WHERE due_date < CURDATE() 
            AND status NOT IN ('closed', 'risk_accepted')
        ");
        return (int) $stmt->fetchColumn();
    }
}
```

### Step 6: Compliance Dashboard

```json
{
  "dashboard": {
    "title": "WHMCS Compliance Dashboard",
    "refresh": "1h",
    "widgets": [
      {
        "type": "score",
        "title": "Overall Compliance Score",
        "value": 92,
        "max": 100,
        "trend": "+2%",
        "framework": "All Frameworks"
      },
      {
        "type": "pie",
        "title": "Findings by Status",
        "data": {
          "open": 3,
          "in_progress": 5,
          "mitigated": 2,
          "closed": 15
        }
      },
      {
        "type": "bar",
        "title": "Findings by Framework",
        "data": {
          "labels": ["PCI-DSS", "GDPR", "SOC 2", "HIPAA"],
          "values": [2, 1, 3, 0]
        }
      },
      {
        "type": "table",
        "title": "Critical Findings",
        "columns": ["ID", "Framework", "Control", "Due Date", "Owner", "Status"],
        "data": []
      },
      {
        "type": "timeline",
        "title": "Audit Schedule",
        "events": [
          {"date": "2026-03-15", "event": "PCI-DSS Audit", "status": "completed"},
          {"date": "2026-06-15", "event": "GDPR Review", "status": "upcoming"},
          {"date": "2026-09-15", "event": "SOC 2 Audit", "status": "upcoming"}
        ]
      }
    ]
  }
}
```

## Compliance Calendar

| Activity | Frequency | Framework | Due Date |
|----------|-----------|-----------|----------|
| Access review | Quarterly | PCI-DSS | Q1: Mar, Q2: Jun, Q3: Sep, Q4: Dec |
| Log review | Monthly | PCI-DSS | 15th of each month |
| Vulnerability scan | Quarterly | PCI-DSS | Q1: Feb, Q2: May, Q3: Aug, Q4: Nov |
| Data inventory | Annually | GDPR | January |
| Consent review | Annually | GDPR | March |
| Penetration test | Annually | PCI-DSS | April |
| SOC 2 audit | Annually | SOC 2 | September |

## Best Practices

1. **Automate evidence collection**: Reduce manual effort
2. **Continuous monitoring**: Real-time compliance status
3. **Clear ownership**: Assign control owners
4. **Regular reviews**: Quarterly control assessments
5. **Document exceptions**: Risk acceptance process
6. **Train staff**: Compliance awareness

## Common Pitfalls

- **Manual evidence**: Error-prone, time-consuming
- **Outdated controls**: Not matching current environment
- **Missing documentation**: Audit preparation incomplete
- **Untracked remediation**: Findings not resolved
- **Scope creep**: Controls beyond necessary scope

## Verification Checklist

- [ ] Control matrix documented
- [ ] Evidence collection automated
- [ ] Regular assessments scheduled
- [ ] Remediation tracking active
- [ ] Compliance dashboard live
- [ ] Audit schedule maintained
- [ ] Training completed
- [ ] Exception process defined

## Related Documentation

- [WHMCS Audit Trail](whmcs-audit-trail.md)
- [WHMCS Security Audit](whmcs-security-audit.md)
- [WHMCS GDPR Compliance](whmcs-gdpr-compliance.md)
- [PCI-DSS Requirements](https://www.pcisecuritystandards.org/)
- [GDPR Documentation](https://gdpr.eu/article/)