# WHMCS Compliance Dashboard DevKit

## Overview

A comprehensive compliance management dashboard for WHMCS that tracks regulatory requirements, manages compliance tasks, monitors adherence to policies, generates compliance reports, and integrates with external compliance services.

## Features

- Compliance requirement tracking
- Policy management
- Audit trail integration
- Compliance reporting
- Risk compliance scoring
- Automated compliance checks
- Document management
- Compliance calendar
- Remediation tracking
- Compliance notifications

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_compliance_requirements` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `requirement_code` VARCHAR(50) NOT NULL,
    `requirement_name` VARCHAR(255) NOT NULL,
    `description` TEXT NULL,
    `regulation` VARCHAR(100) NULL,
    `category` VARCHAR(100) NOT NULL,
    `severity` ENUM('critical', 'high', 'medium', 'low', 'informational') NOT NULL DEFAULT 'medium',
    `compliance_framework` VARCHAR(100) NULL,
    `control_id` VARCHAR(50) NULL,
    `frequency` ENUM('once', 'daily', 'weekly', 'monthly', 'quarterly', 'annually') NOT NULL DEFAULT 'quarterly',
    `evidence_required` TINYINT(1) NOT NULL DEFAULT 0,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_requirement_code` (`requirement_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_compliance_checks` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `requirement_id` INT UNSIGNED NOT NULL,
    `check_type` ENUM('automated', 'manual', 'evidence_review') NOT NULL DEFAULT 'manual',
    `check_method` VARCHAR(255) NULL,
    `check_query` TEXT NULL,
    `expected_result` TEXT NULL,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    CONSTRAINT `fk_check_requirement` FOREIGN KEY (`requirement_id`) REFERENCES `mod_compliance_requirements`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_compliance_results` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `requirement_id` INT UNSIGNED NOT NULL,
    `check_id` INT UNSIGNED NULL,
    `check_date` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `result` ENUM('pass', 'fail', 'warning', 'not_applicable') NOT NULL,
    `actual_value` TEXT NULL,
    `expected_value` TEXT NULL,
    `deviation` TEXT NULL,
    `evidence_files` JSON NULL,
    `notes` TEXT NULL,
    `checked_by` INT UNSIGNED NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_requirement_check` (`requirement_id`, `check_date`),
    INDEX `idx_result_date` (`result`, `check_date`),
    CONSTRAINT `fk_result_requirement` FOREIGN KEY (`requirement_id`) REFERENCES `mod_compliance_requirements`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_compliance_tasks` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `requirement_id` INT UNSIGNED NOT NULL,
    `task_name` VARCHAR(255) NOT NULL,
    `description` TEXT NULL,
    `assigned_to` INT UNSIGNED NULL,
    `due_date` DATE NULL,
    `priority` ENUM('urgent', 'high', 'medium', 'low') NOT NULL DEFAULT 'medium',
    `status` ENUM('pending', 'in_progress', 'completed', 'overdue', 'cancelled') NOT NULL DEFAULT 'pending',
    `resolution_notes` TEXT NULL,
    `completed_at` DATETIME NULL,
    `completed_by` INT UNSIGNED NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_due_status` (`due_date`, `status`),
    INDEX `idx_assigned_status` (`assigned_to`, `status`),
    CONSTRAINT `fk_task_requirement` FOREIGN KEY (`requirement_id`) REFERENCES `mod_compliance_requirements`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Files

### compliance_dashboard.php

```php
<?php
/**
 * WHMCS Compliance Dashboard Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/ComplianceManager.php';
require_once __DIR__ . '/lib/ComplianceChecker.php';
require_once __DIR__ . '/lib/ComplianceReporter.php';

use WHMCS\Module\ComplianceDashboard\ComplianceManager;
use WHMCS\Module\ComplianceDashboard\ComplianceChecker;
use WHMCS\Module\ComplianceDashboard\ComplianceReporter;

/**
 * Activate Module
 */
function whmcs_compliance_dashboard_activate() {
    $manager = new ComplianceManager();
    return $manager->activate();
}

/**
 * Deactivate Module
 */
function whmcs_compliance_dashboard_deactivate() {
    return ['success' => true, 'msg' => 'Compliance Dashboard module deactivated'];
}

/**
 * Module Configuration
 */
function whmcs_compliance_dashboard_config() {
    return [
        'compliance_framework' => [
            'FriendlyName' => 'Primary Framework',
            'Type' => 'dropdown',
            'Options' => [
                'soc2' => 'SOC 2',
                'gdpr' => 'GDPR',
                'pci_dss' => 'PCI DSS',
                'hipaa' => 'HIPAA',
                'custom',
            ],
            'Default' => 'soc2',
        ],
        'auto_checks' => [
            'FriendlyName' => 'Enable Automated Checks',
            'Type' => 'yesno',
            'Description' => 'Run automated compliance checks via cron',
        ],
        'check_frequency' => [
            'FriendlyName' => 'Check Frequency',
            'Type' => 'dropdown',
            'Options' => [
                'daily' => 'Daily',
                'weekly' => 'Weekly',
                'monthly' => 'Monthly',
            ],
            'Default' => 'daily',
        ],
        'critical_threshold' => [
            'FriendlyName' => 'Critical Compliance Threshold',
            'Type' => 'text',
            'Default' => '95',
            'Description' => 'Minimum compliance percentage to avoid alerts',
        ],
        'admin_notification_email' => [
            'FriendlyName' => 'Admin Notification Email',
            'Type' => 'text',
            'Size' => '50',
        ],
    ];
}

/**
 * Get compliance dashboard data
 */
function whmcs_compliance_dashboard_get_data() {
    $manager = new ComplianceManager();
    return $manager->getDashboardData();
}

/**
 * Get compliance score
 */
function whmcs_compliance_dashboard_get_score($framework = null) {
    $manager = new ComplianceManager();
    return $manager->calculateComplianceScore($framework);
}

/**
 * Run compliance check
 */
function whmcs_compliance_dashboard_run_check($requirementId = null) {
    $checker = new ComplianceChecker();
    return $checker->runChecks($requirementId);
}

/**
 * Get compliance report
 */
function whmcs_compliance_dashboard_get_report($startDate, $endDate, $format = 'json') {
    $reporter = new ComplianceReporter();
    return $reporter->generateReport($startDate, $endDate, $format);
}

/**
 * Get compliance tasks
 */
function whmcs_compliance_dashboard_get_tasks($status = null) {
    $manager = new ComplianceManager();
    return $manager->getTasks($status);
}

/**
 * Update compliance task
 */
function whmcs_compliance_dashboard_update_task($taskId, $data) {
    $manager = new ComplianceManager();
    return $manager->updateTask($taskId, $data);
}

// Hooks
add_hook('DailyCronJob', 1, function() {
    if (\App::get_config('compliance_dashboard')['auto_checks']) {
        $checker = new ComplianceChecker();
        $checker->runAutomatedChecks();
    }
});

add_hook('ComplianceCheckReminder', 1, function($params) {
    $manager = new ComplianceManager();
    $manager->sendReminders();
});
```

### lib/ComplianceManager.php

```php
<?php
/**
 * Compliance Manager
 */

namespace WHMCS\Module\ComplianceDashboard;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ComplianceManager {
    
    protected $version = '1.0.0';
    
    public function activate() {
        try {
            $this->createTables();
            $this->initializeRequirements();
            return ['success' => true, 'msg' => 'Compliance Dashboard module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_compliance_requirements` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `requirement_code` VARCHAR(50) NOT NULL,
                `requirement_name` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `regulation` VARCHAR(100) NULL,
                `category` VARCHAR(100) NOT NULL,
                `severity` ENUM('critical', 'high', 'medium', 'low', 'informational') NOT NULL DEFAULT 'medium',
                `frequency` ENUM('once', 'daily', 'weekly', 'monthly', 'quarterly', 'annually') NOT NULL DEFAULT 'quarterly',
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_requirement_code` (`requirement_code`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_compliance_results` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `requirement_id` INT UNSIGNED NOT NULL,
                `check_date` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                `result` ENUM('pass', 'fail', 'warning', 'not_applicable') NOT NULL,
                `actual_value` TEXT NULL,
                `expected_value` TEXT NULL,
                `notes` TEXT NULL,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_compliance_tasks` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `requirement_id` INT UNSIGNED NOT NULL,
                `task_name` VARCHAR(255) NOT NULL,
                `description` VARCHAR(500) NULL,
                `assigned_to` INT UNSIGNED NULL,
                `due_date` DATE NULL,
                `priority` ENUM('urgent', 'high', 'medium', 'low') NOT NULL DEFAULT 'medium',
                `status` ENUM('pending', 'in_progress', 'completed', 'overdue', 'cancelled') NOT NULL DEFAULT 'pending',
                `completed_at` DATETIME NULL,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function initializeRequirements() {
        $requirements = [
            ['code' => 'SEC-001', 'name' => 'Data Encryption', 'category' => 'Security', 'severity' => 'critical', 'regulation' => 'SOC2'],
            ['code' => 'SEC-002', 'name' => 'Access Control Policy', 'category' => 'Security', 'severity' => 'high', 'regulation' => 'SOC2'],
            ['code' => 'SEC-003', 'name' => 'Password Policy', 'category' => 'Security', 'severity' => 'high', 'regulation' => 'SOC2'],
            ['code' => 'PRIV-001', 'name' => 'Data Privacy Policy', 'category' => 'Privacy', 'severity' => 'critical', 'regulation' => 'GDPR'],
            ['code' => 'PRIV-002', 'name' => 'Consent Management', 'category' => 'Privacy', 'severity' => 'high', 'regulation' => 'GDPR'],
            ['code' => 'OPS-001', 'name' => 'Backup Procedures', 'category' => 'Operations', 'severity' => 'high', 'regulation' => 'SOC2'],
            ['code' => 'OPS-002', 'name' => 'Incident Response', 'category' => 'Operations', 'severity' => 'high', 'regulation' => 'SOC2'],
            ['code' => 'OPS-003', 'name' => 'Change Management', 'category' => 'Operations', 'severity' => 'medium', 'regulation' => 'SOC2'],
            ['code' => 'PCI-001', 'name' => 'Cardholder Data Protection', 'category' => 'Payment', 'severity' => 'critical', 'regulation' => 'PCI-DSS'],
            ['code' => 'PCI-002', 'name' => 'Network Security', 'category' => 'Payment', 'severity' => 'critical', 'regulation' => 'PCI-DSS'],
        ];
        
        foreach ($requirements as $req) {
            if (!Capsule::table('mod_compliance_requirements')->where('requirement_code', $req['code'])->exists()) {
                Capsule::table('mod_compliance_requirements')->insert($req);
            }
        }
    }
    
    public function getDashboardData() {
        return [
            'overview' => $this->getOverview(),
            'by_category' => $this->getComplianceByCategory(),
            'by_severity' => $this->getComplianceBySeverity(),
            'recent_checks' => $this->getRecentChecks(),
            'pending_tasks' => $this->getPendingTasks(),
            'upcoming_reviews' => $this->getUpcomingReviews(),
        ];
    }
    
    protected function getOverview() {
        $total = Capsule::table('mod_compliance_requirements')->where('is_active', 1)->count();
        $passing = Capsule::table('mod_compliance_results as cr')
            ->join('mod_compliance_requirements as req', 'cr.requirement_id', '=', 'req.id')
            ->where('req.is_active', 1)
            ->where('cr.result', 'pass')
            ->where('cr.check_date', '>=', Carbon::now()->subDays(30))
            ->count();
        
        $failing = Capsule::table('mod_compliance_results as cr')
            ->join('mod_compliance_requirements as req', 'cr.requirement_id', '=', 'req.id')
            ->where('req.is_active', 1)
            ->where('cr.result', 'fail')
            ->where('cr.check_date', '>=', Carbon::now()->subDays(30))
            ->count();
        
        return [
            'total_requirements' => $total,
            'passing' => $passing,
            'failing' => $failing,
            'pending_reviews' => $total - $passing - $failing,
            'compliance_score' => $total > 0 ? round(($passing / $total) * 100, 2) : 0,
        ];
    }
    
    protected function getComplianceByCategory() {
        $categories = Capsule::table('mod_compliance_requirements')
            ->where('is_active', 1)
            ->groupBy('category')
            ->selectRaw('category, COUNT(*) as total')
            ->get();
        
        $results = [];
        foreach ($categories as $category) {
            $passing = Capsule::table('mod_compliance_results as cr')
                ->join('mod_compliance_requirements as req', 'cr.requirement_id', '=', 'req.id')
                ->where('req.category', $category->category)
                ->where('cr.result', 'pass')
                ->where('cr.check_date', '>=', Carbon::now()->subDays(30))
                ->count();
            
            $results[] = [
                'category' => $category->category,
                'total' => $category->total,
                'passing' => $passing,
                'score' => $category->total > 0 ? round(($passing / $category->total) * 100, 2) : 0,
            ];
        }
        
        return $results;
    }
    
    protected function getComplianceBySeverity() {
        $severities = Capsule::table('mod_compliance_requirements')
            ->where('is_active', 1)
            ->groupBy('severity')
            ->selectRaw('severity, COUNT(*) as total')
            ->get();
        
        $results = [];
        foreach ($severities as $severity) {
            $passing = Capsule::table('mod_compliance_results as cr')
                ->join('mod_compliance_requirements as req', 'cr.requirement_id', '=', 'req.id')
                ->where('req.severity', $severity->severity)
                ->where('cr.result', 'pass')
                ->where('cr.check_date', '>=', Carbon::now()->subDays(30))
                ->count();
            
            $results[] = [
                'severity' => $severity->severity,
                'total' => $severity->total,
                'passing' => $passing,
                'score' => $severity->total > 0 ? round(($passing / $severity->total) * 100, 2) : 0,
            ];
        }
        
        return $results;
    }
    
    protected function getRecentChecks() {
        return Capsule::table('mod_compliance_results as cr')
            ->join('mod_compliance_requirements as req', 'cr.requirement_id', '=', 'req.id')
            ->where('req.is_active', 1)
            ->orderBy('cr.check_date', 'desc')
            ->limit(10)
            ->select('cr.*', 'req.requirement_name', 'req.requirement_code')
            ->get();
    }
    
    public function calculateComplianceScore($framework = null) {
        $query = Capsule::table('mod_compliance_requirements')->where('is_active', 1);
        
        if ($framework) {
            $query->where('regulation', $framework);
        }
        
        $requirements = $query->get();
        $total = $requirements->count();
        
        if ($total === 0) {
            return ['score' => 0, 'total' => 0, 'details' => []];
        }
        
        $passing = 0;
        $details = [];
        
        foreach ($requirements as $req) {
            $latestResult = Capsule::table('mod_compliance_results')
                ->where('requirement_id', $req->id)
                ->orderBy('check_date', 'desc')
                ->first();
            
            $status = $latestResult ? $latestResult->result : 'pending';
            if ($status === 'pass') $passing++;
            
            $details[] = [
                'requirement_code' => $req->requirement_code,
                'requirement_name' => $req->requirement_name,
                'category' => $req->category,
                'severity' => $req->severity,
                'status' => $status,
                'last_check' => $latestResult ? $latestResult->check_date : null,
            ];
        }
        
        return [
            'score' => round(($passing / $total) * 100, 2),
            'total' => $total,
            'passing' => $passing,
            'details' => $details,
        ];
    }
    
    public function getTasks($status = null) {
        $query = Capsule::table('mod_compliance_tasks as t')
            ->join('mod_compliance_requirements as req', 't.requirement_id', '=', 'req.id')
            ->select('t.*', 'req.requirement_name');
        
        if ($status) {
            $query->where('t.status', $status);
        }
        
        return $query->orderBy('t.due_date', 'asc')->get();
    }
    
    public function updateTask($taskId, $data) {
        $updateData = [];
        
        if (isset($data['status'])) {
            $updateData['status'] = $data['status'];
            if ($data['status'] === 'completed') {
                $updateData['completed_at'] = Carbon::now();
            }
        }
        
        if (isset($data['resolution_notes'])) {
            $updateData['resolution_notes'] = $data['resolution_notes'];
        }
        
        if (isset($data['assigned_to'])) {
            $updateData['assigned_to'] = $data['assigned_to'];
        }
        
        Capsule::table('mod_compliance_tasks')->where('id', $taskId)->update($updateData);
        
        return ['success' => true];
    }
    
    public function getPendingTasks() {
        return $this->getTasks('pending');
    }
    
    public function getUpcomingReviews() {
        $next30Days = Carbon::now()->addDays(30)->toDateString();
        
        return Capsule::table('mod_compliance_requirements')
            ->where('is_active', 1)
            ->where('frequency', '!=', 'once')
            ->get()
            ->filter(function($req) use ($next30Days) {
                $lastCheck = Capsule::table('mod_compliance_results')
                    ->where('requirement_id', $req->id)
                    ->max('check_date');
                
                if (!$lastCheck) return true;
                
                $daysSinceCheck = Carbon::parse($lastCheck)->diffInDays(Carbon::now());
                $frequencyDays = $this->frequencyToDays($req->frequency);
                
                return $daysSinceCheck >= ($frequencyDays * 0.8);
            })
            ->take(5)
            ->values();
    }
    
    protected function frequencyToDays($frequency) {
        switch ($frequency) {
            case 'daily': return 1;
            case 'weekly': return 7;
            case 'monthly': return = 30;
            case 'quarterly': return 90;
            case 'annually': return 365;
            default: return 90;
        }
    }
    
    public function sendReminders() {
        $pendingTasks = Capsule::table('mod_compliance_tasks')
            ->whereIn('status', ['pending', 'in_progress'])
            ->where('due_date', '<=', Carbon::now()->addDays(7)->toDateString())
            ->get();
        
        foreach ($pendingTasks as $task) {
            // Send email notification
            $this->sendTaskReminder($task);
        }
    }
    
    protected function sendTaskReminder($task) {
        $adminEmail = \App::get_config('compliance_dashboard')['admin_notification_email'] ?? '';
        
        if ($adminEmail) {
            sendEmail('compliance_reminder', $adminEmail, [
                'task_name' => $task->task_name,
                'due_date' => $task->due_date,
                'priority' => $task->priority,
            ]);
        }
    }
}
```

### lib/ComplianceChecker.php

```php
<?php
/**
 * Compliance Checker
 */

namespace WHMCS\Module\ComplianceDashboard;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ComplianceChecker {
    
    protected $checkMethods = [];
    
    public function runChecks($requirementId = null) {
        if ($requirementId) {
            return $this->runSingleCheck($requirementId);
        }
        
        return $this->runAllChecks();
    }
    
    protected function runSingleCheck($requirementId) {
        $requirement = Capsule::table('mod_compliance_requirements')->where('id', $requirementId)->first();
        
        if (!$requirement) {
            return ['success' => false, 'msg' => 'Requirement not found'];
        }
        
        return $this->executeCheck($requirement);
    }
    
    protected function runAllChecks() {
        $requirements = Capsule::table('mod_compliance_requirements')
            ->where('is_active', 1)
            ->get();
        
        $results = [];
        foreach ($requirements as $req) {
            $results[] = $this->executeCheck($req);
        }
        
        return [
            'success' => true,
            'total_checks' => count($results),
            'passing' => count(array_filter($results, fn($r) => $r['result'] === 'pass')),
            'failing' => count(array_filter($results, fn($r) => $r['result'] === 'fail')),
            'results' => $results,
        ];
    }
    
    protected function executeCheck($requirement) {
        $checkMethod = 'check_' . str_replace('-', '_', $requirement->requirement_code);
        
        if (method_exists($this, $checkMethod)) {
            $checkResult = $this->$checkMethod();
        } else {
            $checkResult = $this->defaultCheck();
        }
        
        $resultId = Capsule::table('mod_compliance_results')->insertGetId([
            'requirement_id' => $requirement->id,
            'check_date' => Carbon::now(),
            'result' => $checkResult['result'],
            'actual_value' => $checkResult['actual_value'] ?? null,
            'expected_value' => $checkResult['expected_value'] ?? null,
            'notes' => $checkResult['notes'] ?? null,
        ]);
        
        return [
            'requirement_code' => $requirement->requirement_code,
            'requirement_name' => $requirement->requirement_name,
            'result' => $checkResult['result'],
            'notes' => $checkResult['notes'] ?? null,
        ];
    }
    
    public function check_SEC_001() {
        // Data Encryption check
        $encryptionsEnabled = Capsule::table('tblconfiguration')
            ->where('setting', 'like', '%encryption%')
            ->exists();
        
        $dbEncryption = Capsule::select("SHOW VARIABLES LIKE 'have_openssl' OR 'have_ssl'%");
        $sslEnabled = !empty($dbEncryption);
        
        return [
            'result' => ($encryptionsEnabled && $sslEnabled) ? 'pass' : 'fail',
            'actual_value' => ($encryptionsEnabled && $sslEnabled) ? 'Enabled' : 'Disabled',
            'expected_value' => 'Enabled',
            'notes' => 'TLS/SSL and encryption settings verification',
        ];
    }
    
    public function check_SEC_002() {
        // Access Control Policy check
        $activeUsers = Capsule::table('tblusers')->where('disabled', 0)->count();
        $adminUsers = Capsule::table('tbladmins')->where('disabled', 0)->count();
        
        return [
            'result' => ($adminUsers > 0 && $adminUsers < 50) ? 'pass' : 'warning',
            'actual_value' => "Active users: $activeUsers, Admin users: $adminUsers",
            'expected_value' => 'Admin users < 50',
            'notes' => 'User account review',
        ];
    }
    
    public function check_SEC_003() {
        // Password Policy check
        $passwordConfig = Capsule::table('tblconfiguration')
            ->where('setting', 'like', '%password%')
            ->pluck('value', 'setting')
            ->toArray();
        
        $minLength = $passwordConfig['password_min_length'] ?? 8;
        $requireSpecial = $passwordConfig['password_requirements'] ?? '';
        
        return [
            'result' => ($minLength >= 8) ? 'pass' : 'warning',
            'actual_value' => "Min length: $minLength",
            'expected_value' => 'Min length: 8',
            'notes' => 'Password policy enforcement',
        ];
    }
    
    public function check_PRIV_001() {
        // Data Privacy Policy check
        $privacyPolicyExists = Capsule::table('mod_pages')->where('type', 'privacy_policy')->exists();
        
        return [
            'result' => $privacyPolicyExists ? 'pass' : 'fail',
            'actual_value' => $privacyPolicyExists ? 'Published' : 'Missing',
            'expected_value' => 'Privacy policy page exists',
            'notes' => 'GDPR Article 12 transparency requirement',
        ];
    }
    
    public function check_PRIV_002() {
        // Consent Management check
        $consentTable = Capsule::table('mod_user_consents')
            ->where('created_at', '>=', Carbon::now()->subDays(30))
            ->count();
        
        return [
            'result' => $consentTable > 0 ? 'pass' : 'warning',
            'actual_value' => "$consentTable consent records in 30 days",
            'expected_value' => 'Consent records exist',
            'notes' => 'GDPR consent tracking',
        ];
    }
    
    public function check_OPS_001() {
        // Backup Procedures check
        $recentBackup = Capsule::table('mod_backup_logs')
            ->where('status', 'success')
            ->where('created_at', '>=', Carbon::now()->subDays(1))
            ->exists();
        
        return [
            'result' => $recentBackup ? 'pass' : 'fail',
            'actual_value' => $recentBackup ? 'Backup within 24h' : 'No recent backup',
            'expected_value' => 'Daily backup verified',
            'notes' => 'Backup procedures verification',
        ];
    }
    
    public function check_OPS_002() {
        // Incident Response check
        $incidentPlan = Capsule::table('mod_documents')
            ->where('category', 'incident_response')
            ->where('status', 'approved')
            ->exists();
        
        return [
            'result' => $incidentPlan ? 'pass' : 'fail',
            'actual_value' => $incidentPlan ? 'Plan documented' : 'Plan missing',
            'expected_value' => 'Incident response plan exists',
            'notes' => 'SOC 2 CC7.3 requirement',
        ];
    }
    
    public function check_PCI_001() {
        // Cardholder Data Protection check
        $noCardStorage = Capsule::table('tblclients')
            ->whereNotNull('card_number')
            ->count() == 0;
        
        return [
            'result' => $noCardStorage ? 'pass' : 'fail',
            'actual_value' => $noCardStorage ? 'No card data stored' : 'Card data found',
            'expected_value' => 'No card data in database',
            'notes' => 'PCI DSS 3.4 requirement',
        ];
    }
    
    public function check_PCI_002() {
        // Network Security check
        $firewallEnabled = Capsule::table('tblconfiguration')
            ->where('setting', 'firewall_enabled')
            ->value('value') ?? 'yes';
        
        return [
            'result' => $firewallEnabled === 'yes' ? 'pass' : 'warning',
            'actual_value' => 'Firewall: ' . $firewallEnabled,
            'expected_value' => 'Firewall enabled',
            'notes' => 'PCI DSS 1.2 requirement',
        ];
    }
    
    public function defaultCheck() {
        return [
            'result' => 'not_applicable',
            'actual_value' => 'No automated check defined',
            'expected_value' => 'Manual review required',
            'notes' => 'This requirement requires manual verification',
        ];
    }
    
    public function runAutomatedChecks() {
        $requirements = Capsule::table('mod_compliance_requirements')
            ->where('is_active', 1)
            ->where(function($query) {
                $query->where('check_type', 'automated')
                    ->orWhereNull('check_type');
            })
            ->get();
        
        foreach ($requirements as $req) {
            $this->executeCheck($req);
        }
    }
}
```

### lib/ComplianceReporter.php

```php
<?php
/**
 * Compliance Reporter
 */

namespace WHMCS\Module\ComplianceDashboard;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;
use TCPDF;

class ComplianceReporter {
    
    public function generateReport($startDate, $endDate, $format = 'json') {
        $data = $this->collectReportData($startDate, $endDate);
        
        if ($format === 'pdf') {
            return $this->generatePdfReport($data, $startDate, $endDate);
        }
        
        return $data;
    }
    
    protected function collectReportData($startDate, $endDate) {
        $results = Capsule::table('mod_compliance_results as cr')
            ->join('mod_compliance_requirements as req', 'cr.requirement_id', '=', 'req.id')
            ->whereBetween('cr.check_date', [$startDate, $endDate])
            ->select('cr.*', 'req.requirement_name', 'req.requirement_code', 'req.category', 'req.severity')
            ->orderBy('cr.check_date', 'desc')
            ->get();
        
        $summary = [
            'total_checks' => $results->count(),
            'passing' => $results->where('result', 'pass')->count(),
            'failing' => $results->where('result', 'fail')->count(),
            'warnings' => $results->where('result', 'warning')->count(),
            'not_applicable' => $results->where('result', 'not_applicable')->count(),
        ];
        
        $summary['compliance_rate'] = $summary['total_checks'] > 0 
            ? round(($summary['passing'] / $summary['total_checks']) * 100, 2) 
            : 0;
        
        return [
            'period' => [
                'start' => $startDate,
                'end' => $endDate,
                'generated_at' => Carbon::now()->toDateTimeString(),
            ],
            'summary' => $summary,
            'by_category' => $this->groupByCategory($results),
            'by_severity' => $this->groupBySeverity($results),
            'detailed_results' => $results,
        ];
    }
    
    protected function groupByCategory($results) {
        return $results->groupBy('category')->map(function($items) {
            return [
                'total' => $items->count(),
                'passing' => $items->where('result', 'pass')->count(),
                'failing' => $items->where('result', 'fail')->count(),
            ];
        })->toArray();
    }
    
    protected function groupBySeverity($results) {
        return $results->groupBy('severity')->map(function($items) {
            return [
                'total' => $items->count(),
                'passing' => $items->where('result', 'pass')->count(),
                'failing' => $items->where('result', 'fail')->count(),
            ];
        })->toArray();
    }
    
    protected function generatePdfReport($data, $startDate, $endDate) {
        // Placeholder for PDF generation
        return [
            'success' => true,
            'format' => 'pdf',
            'filename' => "compliance_report_{$startDate}_{$endDate}.pdf",
        ];
    }
}
```

## API Endpoints

```
GET  /api/v1/compliance/dashboard          - Get dashboard data
GET  /api/v1/compliance/score              - Get compliance score
GET  /api/v1/compliance/score/{framework}  - Get framework-specific score
POST /api/v1/compliance/checks             - Run compliance checks
GET  /api/v1/compliance/requirements      - List all requirements
GET  /api/v1/compliance/results           - Get check results
GET  /api/v1/compliance/tasks             - Get compliance tasks
PUT  /api/v1/compliance/tasks/{id}         - Update task
POST /api/v1/compliance/report            - Generate report
```

## Hooks Integration

```php
// Compliance check on user data export (GDPR)
add_hook('ClientAreaExportData', 1, function($params) {
    // Log data export for compliance
    Capsule::table('mod_compliance_results')->insert([
        'requirement_id' => 1, // PRIV-001
        'result' => 'pass',
        'actual_value' => 'Data export requested',
        'notes' => 'User: ' . $params['user_id'],
    ]);
});

// Alert on compliance failure
add_hook('ComplianceCheckComplete', 1, function($params) {
    if ($params['result'] === 'fail') {
        sendAdminNotification('compliance_failure', $params);
    }
});
```
