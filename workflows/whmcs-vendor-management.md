# WHMCS Vendor Management Workflow

## Purpose

Establish comprehensive vendor management processes for WHMCS to ensure third-party services, integrations, and dependencies are properly evaluated, monitored, and managed throughout their lifecycle. This workflow covers vendor selection, contract management, performance monitoring, and risk assessment.

## Prerequisites

- Vendor inventory system
- Contract management process
- Security assessment framework
- Performance monitoring tools
- Risk management framework

## Workflow Steps

### Step 1: Vendor Categories and Risk Classification

```
┌─────────────────────────────────────────────────────────────┐
│                 Vendor Risk Classification                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CRITICAL (Tier 1)                                         │
│  ├─ Payment processors                                     │
│  ├─ Core infrastructure providers                          │
│  ├─ Domain registrars                                       │
│  └─ SSL certificate providers                              │
│  • Impact: Direct revenue and operations                  │
│  • Review: Annual + on-demand                              │
│  • Backup: Required                                       │
│                                                             │
│  HIGH (Tier 2)                                             │
│  ├─ Email providers                                        │
│  ├─ CDN providers                                          │
│  ├─ SMS gateways                                           │
│  └─ Analytics platforms                                   │
│  • Impact: Service quality                                │
│  • Review: Semi-annual                                    │
│  • Backup: Recommended                                    │
│                                                             │
│  MEDIUM (Tier 3)                                          │
│  ├─ Add-on developers                                      │
│  ├─ Support tools                                         │
│  └─ Marketing integrations                                 │
│  • Impact: User experience                                │
│  • Review: Annual                                          │
│                                                             │
│  LOW (Tier 4)                                             │
│  ├─ Non-essential integrations                             │
│  └─ Testing tools                                         │
│  • Impact: Minimal                                         │
│  • Review: As needed                                       │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Vendor Assessment Process

```markdown
## WHMCS Vendor Security Assessment Checklist

### 1. General Information
- Vendor name:
- Service provided:
- Contract value:
- Contract term:
- Renewal date:

### 2. Security Assessment

#### Data Security
- [ ] SOC 2 Type II report available
- [ ] ISO 27001 certification
- [ ] Penetration test results
- [ ] Data encryption at rest
- [ ] Data encryption in transit
- [ ] Data retention policy documented
- [ ] Data deletion procedures confirmed

#### Access Control
- [ ] User access controls defined
- [ ] API key management process
- [ ] Multi-factor authentication available
- [ ] Session management described
- [ ] IP restrictions supported

#### Compliance
- [ ] GDPR compliant
- [ ] PCI-DSS compliant (if handling payments)
- [ ] Privacy policy reviewed
- [ ] DPA signed
- [ ] Sub-processor list provided

#### Business Continuity
- [ ] SLA defined
- [ ] Uptime guarantee
- [ ] Redundancy explained
- [ ] Disaster recovery plan
- [ ] Exit strategy documented

#### Financial Stability
- [ ] Financial statements reviewed
- [ ] Company history verified
- [ ] Insurance coverage confirmed
- [ ] Exit costs evaluated

### 3. Risk Assessment

| Risk Category | Risk Level | Mitigation |
|--------------|------------|------------|
| Data breach | | |
| Service discontinuation | | |
| Price increases | | |
| Lock-in | | |
| Compliance violation | | |

### 4. Approval

| Role | Name | Decision | Date |
|------|------|----------|------|
| Requester | | | |
| Security Review | | | |
| Legal Review | | | |
| Finance Review | | | |
| Final Approval | | | |
```

### Step 3: Vendor Management Database

```sql
-- Vendor management database schema
CREATE TABLE IF NOT EXISTS `mod_vendor_management` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `vendor_name` VARCHAR(255) NOT NULL,
    `vendor_type` ENUM('payment', 'infrastructure', 'addon', 'service', 'other') NOT NULL,
    `tier` ENUM('critical', 'high', 'medium', 'low') NOT NULL DEFAULT 'medium',
    `website` VARCHAR(500) NULL,
    `primary_contact` VARCHAR(255) NULL,
    `contact_email` VARCHAR(255) NULL,
    `contract_value` DECIMAL(10,2) NULL,
    `contract_start` DATE NULL,
    `contract_end` DATE NULL,
    `renewal_notice_days` INT NOT NULL DEFAULT 60,
    `sla_uptime` VARCHAR(50) NULL,
    `soc2_compliant` TINYINT(1) NOT NULL DEFAULT 0,
    `gdpr_compliant` TINYINT(1) NOT NULL DEFAULT 0,
    `pci_compliant` TINYINT(1) NOT NULL DEFAULT 0,
    `security_contact` VARCHAR(255) NULL,
    `last_assessment_date` DATE NULL,
    `next_assessment_date` DATE NULL,
    `risk_score` INT NULL,
    `status` ENUM('active', 'inactive', 'pending', 'terminated') NOT NULL DEFAULT 'pending',
    `notes` TEXT NULL,
    PRIMARY KEY (`id`),
    INDEX `idx_tier` (`tier`),
    INDEX `idx_status` (`status`),
    INDEX `idx_renewal` (`contract_end`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Vendor API credentials (encrypted)
CREATE TABLE IF NOT EXISTS `mod_vendor_credentials` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `vendor_id` INT UNSIGNED NOT NULL,
    `credential_name` VARCHAR(100) NOT NULL,
    `encrypted_value` TEXT NOT NULL,
    `environment` ENUM('production', 'staging', 'development') NOT NULL DEFAULT 'production',
    `last_rotated` DATE NULL,
    `next_rotation` DATE NULL,
    `rotation_frequency_days` INT NOT NULL DEFAULT 90,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `created_by` INT UNSIGNED NULL,
    PRIMARY KEY (`id`),
    INDEX `idx_vendor` (`vendor_id`),
    FOREIGN KEY (`vendor_id`) REFERENCES `mod_vendor_management`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Vendor performance metrics
CREATE TABLE IF NOT EXISTS `mod_vendor_performance` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `vendor_id` INT UNSIGNED NOT NULL,
    `metric_date` DATE NOT NULL,
    `uptime_percent` DECIMAL(5,2) NULL,
    `response_time_ms` INT NULL,
    `error_rate_percent` DECIMAL(5,2) NULL,
    `incidents_count` INT NOT NULL DEFAULT 0,
    `support_tickets` INT NOT NULL DEFAULT 0,
    `notes` TEXT NULL,
    PRIMARY KEY (`id`),
    INDEX `idx_vendor_date` (`vendor_id`, `metric_date`),
    FOREIGN KEY (`vendor_id`) REFERENCES `mod_vendor_management`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Step 4: Vendor Performance Monitoring

```php
<?php
// /opt/scripts/vendor_monitoring.php

class VendorMonitor {
    private $pdo;
    
    public function __construct() {
        $this->pdo = new PDO('mysql:host=localhost', 'whmcs', 'password');
    }
    
    public function checkVendorUptime(string $vendorEndpoint): array {
        $startTime = microtime(true);
        
        $ch = curl_init($vendorEndpoint);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 10,
            CURLOPT_SSL_VERIFYPEER => true
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);
        
        $responseTime = (microtime(true) - $startTime) * 1000;
        
        return [
            'available' => ($httpCode >= 200 && $httpCode < 400),
            'http_code' => $httpCode,
            'response_time_ms' => round($responseTime, 2),
            'error' => $error
        ];
    }
    
    public function recordPerformance(int $vendorId, array $metrics): void {
        $stmt = $this->pdo->prepare("
            INSERT INTO mod_vendor_performance (
                vendor_id, metric_date, uptime_percent,
                response_time_ms, error_rate_percent,
                incidents_count, support_tickets
            ) VALUES (?, CURDATE(), ?, ?, ?, ?, ?)
        ");
        
        $stmt->execute([
            $vendorId,
            $metrics['uptime'] ?? null,
            $metrics['response_time'] ?? null,
            $metrics['error_rate'] ?? null,
            $metrics['incidents'] ?? 0,
            $metrics['tickets'] ?? 0
        ]);
    }
    
    public function checkCriticalVendors(): array {
        $stmt = $this->pdo->query("
            SELECT * FROM mod_vendor_management 
            WHERE tier = 'critical' AND status = 'active'
        ");
        
        $alerts = [];
        
        foreach ($stmt->fetchAll(PDO::FETCH_ASSOC) as $vendor) {
            $endpoint = $vendor['health_check_url'] ?? null;
            if (!$endpoint) continue;
            
            $result = $this->checkVendorUptime($endpoint);
            
            if (!$result['available']) {
                $alerts[] = [
                    'vendor' => $vendor['vendor_name'],
                    'issue' => 'Service unavailable',
                    'http_code' => $result['http_code'],
                    'time' => date('Y-m-d H:i:s')
                ];
                
                // Page on-call
                $this->alertVendorIssue($vendor, $result);
            }
            
            if ($result['response_time_ms'] > 1000) {
                $alerts[] = [
                    'vendor' => $vendor['vendor_name'],
                    'issue' => 'Slow response',
                    'response_time' => $result['response_time_ms'],
                    'time' => date('Y-m-d H:i:s')
                ];
            }
        }
        
        return $alerts;
    }
    
    public function getVendorHealthReport(): array {
        $stmt = $this->pdo->query("
            SELECT 
                v.vendor_name,
                v.tier,
                v.sla_uptime,
                p.metric_date,
                p.uptime_percent,
                p.response_time_ms,
                p.error_rate_percent,
                (SELECT COUNT(*) FROM mod_vendor_performance 
                 WHERE vendor_id = v.id AND incidents_count > 0 
                 AND metric_date > DATE_SUB(CURDATE(), INTERVAL 30 DAY)) as incidents_30d
            FROM mod_vendor_management v
            LEFT JOIN mod_vendor_performance p ON v.id = p.vendor_id 
                AND p.metric_date = CURDATE()
            WHERE v.status = 'active'
            ORDER BY FIELD(v.tier, 'critical', 'high', 'medium', 'low'), v.vendor_name
        ");
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    private function alertVendorIssue(array $vendor, array $result): void {
        $message = "ALERT: Vendor {$vendor['vendor_name']} issue detected";
        
        logActivity($message);
        
        // Send alert to monitoring system
        // Integration with PagerDuty, Slack, etc.
    }
}

// Run monitoring
$monitor = new VendorMonitor();

// Check critical vendors every 5 minutes
$alerts = $monitor->checkCriticalVendors();
if (!empty($alerts)) {
    // Process alerts
}
```

### Step 5: Vendor Contract Management

```php
<?php
// /opt/scripts/contract_management.php

class ContractManager {
    private $pdo;
    
    public function __construct() {
        $this->pdo = new PDO('mysql:host=localhost', 'whmcs', 'password');
    }
    
    public function getUpcomingRenewals(int $days = 60): array {
        return $this->pdo->query("
            SELECT 
                v.vendor_name,
                v.contract_end,
                v.contract_value,
                v.primary_contact,
                v.contact_email,
                DATEDIFF(v.contract_end, CURDATE()) as days_until_renewal
            FROM mod_vendor_management v
            WHERE v.status = 'active'
            AND v.contract_end BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL $days DAY)
            ORDER BY days_until_renewal ASC
        ")->fetchAll(PDO::FETCH_ASSOC);
    }
    
    public function getExpiringCredentials(int $days = 30): array {
        return $this->pdo->query("
            SELECT 
                v.vendor_name,
                c.credential_name,
                c.environment,
                c.next_rotation,
                DATEDIFF(c.next_rotation, CURDATE()) as days_until_rotation
            FROM mod_vendor_credentials c
            JOIN mod_vendor_management v ON c.vendor_id = v.id
            WHERE c.next_rotation BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL $days DAY)
            ORDER BY days_until_rotation ASC
        ")->fetchAll(PDO::FETCH_ASSOC);
    }
    
    public function rotateCredential(int $credentialId): bool {
        // Implementation for credential rotation
        // Would call vendor API to generate new credentials
        
        return true;
    }
    
    public function createRenewalTask(int $vendorId): void {
        $vendor = $this->getVendor($vendorId);
        
        // Create task in project management system
        // or add to calendar
    }
    
    private function getVendor(int $vendorId): array {
        $stmt = $this->pdo->prepare("SELECT * FROM mod_vendor_management WHERE id = ?");
        $stmt->execute([$vendorId]);
        return $stmt->fetch(PDO::FETCH_ASSOC);
    }
}

// Generate renewal alerts
$manager = new ContractManager();

// 60-day renewal alerts
$renewals = $manager->getUpcomingRenewals(60);
foreach ($renewals as $renewal) {
    if ($renewal['days_until_renewal'] <= 60) {
        // Send notification
        mail(
            'contracts@example.com',
            "Vendor Renewal: {$renewal['vendor_name']}",
            "Contract for {$renewal['vendor_name']} expires on {$renewal['contract_end']}"
        );
    }
}

// 30-day credential rotation alerts
$credentials = $manager->getExpiringCredentials(30);
foreach ($credentials as $cred) {
    if ($cred['days_until_rotation'] <= 30) {
        // Queue rotation task
    }
}
```

### Step 6: Vendor Risk Assessment

```markdown
## Vendor Risk Assessment Report

### Vendor: [Name]
### Assessment Date: YYYY-MM-DD
### Assessor: [Name]

#### 1. Service Criticality
| Factor | Score (1-5) | Notes |
|--------|-------------|-------|
| Revenue impact | | |
| Customer data sensitivity | | |
| System dependencies | | |
|替换成本 | | |

**Total Score:** /20

#### 2. Security Posture
| Factor | Score (1-5) | Notes |
|--------|-------------|-------|
| Security certifications | | |
| Encryption practices | | |
| Access controls | | |
| Incident history | | |

**Total Score:** /20

#### 3. Business Risk
| Factor | Score (1-5) | Notes |
|--------|-------------|-------|
| Financial stability | | |
| Market position | | |
| Exit complexity | | |
| Contract terms | | |

**Total Score:** /20

#### Overall Risk Score: /60

| Score Range | Risk Level | Action |
|-------------|------------|--------|
| 0-20 | Low | Accept with monitoring |
| 21-35 | Medium | Review and mitigate |
| 36-50 | High | Require mitigation plan |
| 51-60 | Critical | Reject or exit |

#### Mitigation Requirements
1.
2.

#### Sign-off
- Security:
- Legal:
- Finance:
- IT:
```

## Vendor Onboarding Checklist

```
Pre-onboarding:
□ Security assessment completed
□ Financial review completed
□ Legal review (DPA, SLA) completed
□ Technical evaluation completed
□ References checked

Contract:
□ Contract negotiated and signed
□ DPA signed
□ SLA defined
□ Exit provisions included
□ Data handling terms defined

Technical Integration:
□ API credentials issued
□ Webhook endpoints configured
□ Monitoring configured
□ Backup plan documented

Post-onboarding:
□ Team trained on vendor integration
□ Runbook created
□ Emergency contacts documented
□ Regular review scheduled
```

## Vendor Review Schedule

| Vendor Tier | Review Frequency | Review Team | Key Metrics |
|-------------|-----------------|--------------|-------------|
| Critical | Monthly | CISO + VP Eng | Uptime, incidents, response time |
| High | Quarterly | IT Director + Security | Performance, compliance |
| Medium | Semi-annual | IT Manager | General health |
| Low | Annual | IT Manager | Validity check |

## Best Practices

1. **Single source of truth**: Central vendor database
2. **Risk-based approach**: Focus on critical vendors
3. **Continuous monitoring**: Real-time performance tracking
4. **Regular reviews**: Scheduled assessments
5. **Exit planning**: Always plan for termination
6. **Documentation**: Keep all vendor interactions logged

## Common Pitfalls

- **No vendor inventory**: Unknown dependencies
- **Outdated assessments**: Security posture changed
- **Contract gaps**: Missing DPAs or SLAs
- **Credential rotation**: Expired API keys
- **No exit plan**: Dependent on vendor

## Verification Checklist

- [ ] Vendor inventory complete
- [ ] Security assessments done
- [ ] Contracts reviewed and signed
- [ ] Performance monitoring active
- [ ] Renewal alerts configured
- [ ] Exit strategy documented
- [ ] Incident response plan in place
- [ ] Regular reviews scheduled

## Related Documentation

- [WHMCS Security Audit](whmcs-security-audit.md)
- [WHMCS Compliance Reporting](whmcs-compliance-reporting.md)
- [WHMCS Integration Testing](whmcs-integration-testing-workflow.md)