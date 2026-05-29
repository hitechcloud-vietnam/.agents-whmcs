# WHMCS Risk Assessment DevKit

## Overview

A comprehensive business risk assessment system for WHMCS that evaluates operational, financial, compliance, and reputational risks, provides risk scoring, generates mitigation plans, and monitors risk indicators in real-time.

## Features

- Multi-dimensional risk scoring
- Risk category management
- Quantitative risk analysis
- Qualitative risk assessment
- Risk mitigation tracking
- Risk indicator monitoring
- Compliance risk tracking
- Financial risk assessment
- Operational risk evaluation
- Risk reporting and alerts

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_risk_categories` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `category_name` VARCHAR(255) NOT NULL,
    `category_code` VARCHAR(50) NOT NULL,
    `description` TEXT NULL,
    `parent_id` INT UNSIGNED NULL,
    `weight` DECIMAL(3,2) NOT NULL DEFAULT 1.00,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `sort_order` INT UNSIGNED NOT NULL DEFAULT 0,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_category_code` (`category_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_risk_assessments` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `risk_id` VARCHAR(64) NOT NULL,
    `category_id` INT UNSIGNED NOT NULL,
    `risk_name` VARCHAR(255) NOT NULL,
    `risk_description` TEXT NULL,
    `risk_type` ENUM('operational', 'financial', 'compliance', 'reputational', 'strategic', 'technical') NOT NULL DEFAULT 'operational',
    `likelihood` DECIMAL(3,2) NOT NULL DEFAULT 0.00,
    `impact` DECIMAL(3,2) NOT NULL DEFAULT 0.00,
    `risk_score` DECIMAL(6,2) NOT NULL DEFAULT 0.00,
    `risk_level` ENUM('low', 'medium', 'high', 'critical') NOT NULL DEFAULT 'low',
    `inherent_risk_score` DECIMAL(6,2) NOT NULL DEFAULT 0.00,
    `residual_risk_score` DECIMAL(6,2) NOT NULL DEFAULT 0.00,
    `owner_id` INT UNSIGNED NULL,
    `status` ENUM('identified', 'assessed', 'mitigating', 'monitoring', 'resolved', 'accepted') NOT NULL DEFAULT 'identified',
    `identified_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `assessed_at` DATETIME NULL,
    `last_reviewed_at` DATETIME NULL,
    `next_review_date` DATE NULL,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_risk_id` (`risk_id`),
    INDEX `idx_category_status` (`category_id`, `status`),
    INDEX `idx_risk_level` (`risk_level`),
    CONSTRAINT `fk_assessment_category` FOREIGN KEY (`category_id`) REFERENCES `mod_risk_categories`(`id`) ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_risk_controls` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `risk_id` VARCHAR(64) NOT NULL,
    `control_name` VARCHAR(255) NOT NULL,
    `control_description` TEXT NULL,
    `control_type` ENUM('preventive', 'detective', 'corrective', 'compensating') NOT NULL DEFAULT 'preventive',
    `implementation_status` ENUM('planned', 'in_progress', 'implemented', 'ineffective') NOT NULL DEFAULT 'planned',
    `effectiveness_score` DECIMAL(5,2) DEFAULT 0.00,
    `cost` DECIMAL(12,2) NULL,
    `start_date` DATE NULL,
    `completion_date` DATE NULL,
    `control_owner` INT UNSIGNED NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_risk_id` (`risk_id`),
    INDEX `idx_status` (`implementation_status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_risk_indicators` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `risk_id` VARCHAR(64) NOT NULL,
    `indicator_name` VARCHAR(255) NOT NULL,
    `indicator_type` ENUM('lagging', 'leading') NOT NULL DEFAULT 'leading',
    `measurement_unit` VARCHAR(50) NULL,
    `current_value` DECIMAL(12,4) NULL,
    `threshold_warning` DECIMAL(12,4) NULL,
    `threshold_critical` DECIMAL(12,4) NULL,
    `target_value` DECIMAL(12,4) NULL,
    `value_direction` ENUM('below_better', 'above_better') NOT NULL DEFAULT 'below_better',
    `last_measured_at` DATETIME NULL,
    `alert_enabled` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_risk_indicator` (`risk_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_risk_incidents` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `risk_id` VARCHAR(64) NOT NULL,
    `incident_date` DATETIME NOT NULL,
    `severity` ENUM('minor', 'moderate', 'major', 'severe') NOT NULL DEFAULT 'moderate',
    `description` TEXT NOT NULL,
    `financial_impact` DECIMAL(12,2) DEFAULT 0.00,
    `operational_impact` TEXT NULL,
    `response_status` ENUM('detected', 'responding', 'contained', 'resolved', 'reviewed') NOT NULL DEFAULT 'detected',
    `lessons_learned` TEXT NULL,
    `recorded_by` INT UNSIGNED NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_risk_date` (`risk_id`, `incident_date`),
    INDEX `idx_severity` (`severity`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Files

### risk_assessment.php

```php
<?php
/**
 * WHMCS Risk Assessment Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/RiskManager.php';
require_once __DIR__ . '/lib/RiskCalculator.php';
require_once __DIR__ . '/lib/RiskIndicatorMonitor.php';

use WHMCS\Module\RiskAssessment\RiskManager;
use WHMCS\Module\RiskAssessment\RiskCalculator;
use WHMCS\Module\RiskAssessment\RiskIndicatorMonitor;

function whmcs_risk_assessment_activate() {
    $manager = new RiskManager();
    return $manager->activate();
}

function whmcs_risk_assessment_deactivate() {
    return ['success' => true, 'msg' => 'Risk Assessment module deactivated'];
}

function whmcs_risk_assessment_config() {
    return [
        'risk_threshold_low' => [
            'FriendlyName' => 'Low Risk Threshold',
            'Type' => 'text',
            'Default' => '25',
        ],
        'risk_threshold_high' => [
            'FriendlyName' => 'High Risk Threshold',
            'Type' => 'text',
            'Default' => '65',
        ],
        'auto_monitoring' => [
            'FriendlyName' => 'Enable Auto Monitoring',
            'Type' => 'yesno',
        ],
        'alert_email' => [
            'FriendlyName' => 'Alert Email',
            'Type' => 'text',
            'Size' => '50',
        ],
        'monitoring_frequency' => [
            'FriendlyName' => 'Monitoring Frequency',
            'Type' => 'dropdown',
            'Options' => [
                'hourly' => 'Hourly',
                'daily' => 'Daily',
                'weekly' => 'Weekly',
            ],
            'Default' => 'daily',
        ],
    ];
}

/**
 * Get overall risk score
 */
function whmcs_risk_assessment_get_score() {
    $calculator = new RiskCalculator();
    return $calculator->calculateOverallRisk();
}

/**
 * Add new risk
 */
function whmcs_risk_assessment_add_risk($data) {
    $manager = new RiskManager();
    return $manager->addRisk($data);
}

/**
 * Get risk by ID
 */
function whmcs_risk_assessment_get_risk($riskId) {
    $manager = new RiskManager();
    return $manager->getRisk($riskId);
}

/**
 * Update risk assessment
 */
function whmcs_risk_assessment_update_risk($riskId, $data) {
    $manager = new RiskManager();
    return $manager->updateRisk($riskId, $data);
}

/**
 * Get risks by category
 */
function whmcs_risk_assessment_get_by_category($categoryId = null) {
    $manager = new RiskManager();
    return $manager->getRisksByCategory($categoryId);
}

/**
 * Get risk dashboard
 */
function whmcs_risk_assessment_get_dashboard() {
    $manager = new RiskManager();
    $calculator = new RiskCalculator();
    
    return [
        'overview' => $calculator->getOverview(),
        'by_category' => $calculator->getRiskByCategory(),
        'by_level' => $calculator->getRiskByLevel(),
        'top_risks' => $manager->getTopRisks(10),
        'recent_incidents' => $manager->getRecentIncidents(5),
        'pending_reviews' => $manager->getPendingReviews(),
    ];
}

// Hooks
add_hook('DailyCronJob', 1, function() {
    $monitor = new RiskIndicatorMonitor();
    $monitor->checkAllIndicators();
    $monitor->generateAlerts();
});

add_hook('InvoicePaymentFailed', 1, function($params) {
    $manager new RiskManager();
    $manager->logRiskIndicator('financial', 'invoice_failed', $params['amount'] ?? 0);
});

add_hook('ServiceSuspended', 1, function($params) {
    $manager = new RiskManager();
    $manager->updateRiskIndicator('operational', 'service_suspension', 1);
});
```

### lib/RiskManager.php

```php
<?php
/**
 * Risk Manager
 */

namespace WHMCS\Module\RiskAssessment;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class RiskManager {
    
    public function activate() {
        try {
            $this->createTables();
            $this->initializeCategories();
            return ['success' => true, 'msg' => 'Risk Assessment module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_risk_categories` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `category_name` VARCHAR(255) NOT NULL,
                `category_code` VARCHAR(50) NOT NULL,
                `description` TEXT NULL,
                `parent_id` INT UNSIGNED NULL,
                `weight` DECIMAL(3,2) NOT NULL DEFAULT 1.00,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                `sort_order` INT UNSIGNED NOT NULL DEFAULT 0,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_category_code` (`category_code`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_risk_assessments` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `risk_id` VARCHAR(64) NOT NULL UNIQUE,
                `category_id` INT UNSIGNED NOT NULL,
                `risk_name` VARCHAR(255) NOT NULL,
                `risk_description` TEXT NULL,
                `risk_type` ENUM('operational', 'financial', 'compliance', 'reputational', 'strategic', 'technical') NOT NULL DEFAULT 'operational',
                `likelihood` DECIMAL(3,2) NOT NULL DEFAULT 0.00,
                `impact` DECIMAL(3,2) NOT NULL DEFAULT 0.00,
                `risk_score` DECIMAL(6,2) NOT NULL DEFAULT 0.00,
                `risk_level` ENUM('low', 'medium', 'high', 'critical') NOT NULL DEFAULT 'low',
                `inherent_risk_score` DECIMAL(6,2) NOT NULL DEFAULT 0.00,
                `residual_risk_score` DECIMAL(6,2) NOT NULL DEFAULT 0.00,
                `owner_id` INT UNSIGNED NULL,
                `status` ENUM('identified', 'assessed', 'mitigating', 'monitoring', 'resolved', 'accepted') NOT NULL DEFAULT 'identified',
                `identified_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                `assessed_at` DATETIME NULL,
                `next_review_date` DATE NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_risk_controls` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `risk_id` VARCHAR(64) NOT NULL,
                `control_name` VARCHAR(255) NOT NULL,
                `control_description` TEXT NULL,
                `control_type` ENUM('preventive', 'detective', 'corrective', 'compensating') NOT NULL DEFAULT 'preventive',
                `implementation_status` ENUM('planned', 'in_progress', 'implemented', 'ineffective') NOT NULL DEFAULT 'planned',
                `effectiveness_score` DECIMAL(5,2) DEFAULT 0.00,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_risk_indicators` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `risk_id` VARCHAR(64) NOT NULL,
                `indicator_name` VARCHAR(255) NOT NULL,
                `indicator_type` ENUM('lagging', 'leading') NOT NULL DEFAULT 'leading',
                `current_value` DECIMAL(12,4) NULL,
                `threshold_warning` DECIMAL(12,4) NULL,
                `threshold_critical` DECIMAL(12,4) NULL,
                `last_measured_at` DATETIME NULL,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_risk_incidents` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `risk_id` VARCHAR(64) NOT NULL,
                `incident_date` DATETIME NOT NULL,
                `severity` ENUM('minor', 'moderate', 'major', 'severe') NOT NULL DEFAULT 'moderate',
                `description` TEXT NOT NULL,
                `financial_impact` DECIMAL(12,2) DEFAULT 0.00,
                `response_status` ENUM('detected', 'responding', 'contained', 'resolved', 'reviewed') NOT NULL DEFAULT 'detected',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function initializeCategories() {
        $categories = [
            ['code' => 'FIN', 'name' => 'Financial Risk', 'weight' => 1.5, 'sort_order' => 1],
            ['code' => 'OPS', 'name' => 'Operational Risk', 'weight' => 1.0, 'sort_order' => 2],
            ['code' => 'COMP', 'name' => 'Compliance Risk', 'weight' => 1.5, 'sort_order' => 3],
            ['code' => 'TECH', 'name' => 'Technical Risk', 'weight' => 1.2, 'sort_order' => 4],
            ['code' => 'REP', 'name' => 'Reputational Risk', 'weight' => 1.0, 'sort_order' => 5],
            ['code' => 'STR', 'name' => 'Strategic Risk', 'weight' => 0.8, 'sort_order' => 6],
        ];
        
        foreach ($categories as $cat) {
            if (!Capsule::table('mod_risk_categories')->where('category_code', $cat['code'])->exists()) {
                Capsule::table('mod_risk_categories')->insert([
                    'category_name' => $cat['name'],
                    'category_code' => $cat['code'],
                    'weight' => $cat['weight'],
                    'sort_order' => $cat['sort_order'],
                    'is_active' => 1,
                ]);
            }
        }
    }
    
    protected function generateRiskId() {
        return 'RSK-' . date('Y') . '-' . strtoupper(substr(uniqid(), -8));
    }
    
    public function addRisk($data) {
        $riskId = $this->generateRiskId();
        
        $riskScore = $this->calculateRiskScore($data['likelihood'], $data['impact']);
        $riskLevel = $this->determineRiskLevel($riskScore);
        
        $categoryId = Capsule::table('mod_risk_categories')
            ->where('category_code', $data['category_code'])
            ->value('id');
        
        $riskData = [
            'risk_id' => $riskId,
            'category_id' => $categoryId,
            'risk_name' => $data['name'],
            'risk_description' => $data['description'] ?? null,
            'risk_type' => $data['type'] ?? 'operational',
            'likelihood' => $data['likelihood'],
            'impact' => $data['impact'],
            'risk_score' => $riskScore,
            'risk_level' => $riskLevel,
            'inherent_risk_score' => $riskScore,
            'status' => 'identified',
            'identified_at' => Carbon::now(),
            'assessed_at' => Carbon::now(),
            'next_review_date' => $data['review_date'] ?? Carbon::now()->addMonths(3)->toDateString(),
        ];
        
        $riskId = Capsule::table('mod_risk_assessments')->insertGetId($riskData);
        
        return [
            'success' => true,
            'risk_id' => $riskId,
            'generated_id' => $riskData['risk_id'],
            'risk_score' => $riskScore,
            'risk_level' => $riskLevel,
        ];
    }
    
    protected function calculateRiskScore($likelihood, $impact) {
        // Standard risk matrix: Score = Likelihood x Impact
        // Both scaled 1-10
        return ($likelihood * $impact) / 10 * 100;
    }
    
    protected function determineRiskLevel($score) {
        if ($score >= 75) return 'critical';
        if ($score >= 50) return 'high';
        if ($score >= 25) return 'medium';
        return 'low';
    }
    
    public function getRisk($riskId) {
        $risk = Capsule::table('mod_risk_assessments')
            ->where('risk_id', $riskId)
            ->first();
        
        if (!$risk) {
            return null;
        }
        
        $category = Capsule::table('mod_risk_categories')
            ->where('id', $risk->category_id)
            ->first();
        
        $controls = Capsule::table('mod_risk_controls')
            ->where('risk_id', $riskId)
            ->get();
        
        $indicators = Capsule::table('mod_risk_indicators')
            ->where('risk_id', $riskId)
            ->get();
        
        return [
            'risk' => $risk,
            'category' => $category,
            'controls' => $controls,
            'indicators' => $indicators,
        ];
    }
    
    public function updateRisk($riskId, $data) {
        $risk = Capsule::table('mod_risk_assessments')->where('risk_id', $riskId)->first();
        
        if (!$risk) {
            return ['success' => false, 'msg' => 'Risk not found'];
        }
        
        $updateData = ['updated_at' => Carbon::now()];
        
        if (isset($data['likelihood'])) {
            $updateData['likelihood'] = $data['likelihood'];
        }
        if (isset($data['impact'])) {
            $updateData['impact'] = $data['impact'];
        }
        
        // Recalculate risk score
        $likelihood = $data['likelihood'] ?? $risk->likelihood;
        $impact = $data['impact'] ?? $risk->impact;
        $updateData['risk_score'] = $this->calculateRiskScore($likelihood, $impact);
        $updateData['risk_level'] = $this->determineRiskLevel($updateData['risk_score']);
        
        if (isset($data['status'])) {
            $updateData['status'] = $data['status'];
            if ($data['status'] === 'resolved') {
                $updateData['resolved_at'] = Carbon::now();
            }
        }
        
        Capsule::table('mod_risk_assessments')
            ->where('risk_id', $riskId
            ->update($updateData);
        
        // Recalculate residual risk based on controls
        $this->updateResidualRisk($riskId);
        
        return [
            'success' => true,
            'msg' => 'Risk updated',
            'new_risk_score' => $updateData['risk_score'],
            'new_risk_level' => $updateData['risk_level'],
        ];
    }
    
    protected function updateResidualRisk($riskId) {
        $risk = Capsule::table('mod_risk_assessments')->where('risk_id', $riskId)->first();
        $controls = Capsule::table('mod_risk_controls')
            ->where('risk_id', $riskId)
            ->where('implementation_status', 'implemented')
            ->get();
        
        $mitigationFactor = 0;
        foreach ($controls as $control) {
            $mitigationFactor += $control->effectiveness_score / 100;
        }
        
        $residualRisk = $risk->inherent_risk_score * (1 - min(0.9, $mitigationFactor));
        
        Capsule::table('mod_risk_assessments')
            ->where('risk_id', $riskId)
            ->update(['residual_risk_score' => $residualRisk]);
    }
    
    public function getRisksByCategory($categoryId = null) {
        $query = Capsule::table('mod_risk_assessments as r')
            ->join('mod_risk_categories as c', 'r.category_id', '=', 'c.id')
            ->where('r.is_active', 1);
        
        if ($categoryId) {
            $query->where('c.id', $categoryId);
        }
        
        return $query->select('r.*', 'c.category_name', 'c.category_code')
            ->orderBy('r.risk_score', 'desc')
            ->get();
    }
    
    public function getTopRisks($limit = 10) {
        return Capsule::table('mod_risk_assessments')
            ->where('is_active', 1)
            ->whereIn('status', ['identified', 'assessed', 'mitigating'])
            ->orderBy('risk_score', 'desc')
            ->limit($limit)
            ->get();
    }
    
    public function getRecentIncidents($limit = 5) {
        return Capsule::table('mod_risk_incidents as i')
            ->join('mod_risk_assessments as r', 'i.risk_id', '=', 'r.risk_id')
            ->select('i.*', 'r.risk_name')
            ->orderBy('i.incident_date', 'desc')
            ->limit($limit)
            ->get();
    }
    
    public function getPendingReviews() {
        return Capsule::table('mod_risk_assessments')
            ->where('is_active', 1)
            ->where('next_review_date', '<=', Carbon::now()->addDays(7)->toDateString())
            ->orderBy('next_review_date', 'asc')
            ->limit(10)
            ->get();
    }
    
    public function addControl($riskId, $data) {
        $controlId = Capsule::table('mod_risk_controls')->insertGetId([
            'risk_id' => $riskId,
            'control_name' => $data['name'],
            'control_description' => $data['description'] ?? null,
            'control_type' => $data['type'] ?? 'preventive',
            'implementation_status' => 'planned',
        ]);
        
        $this->updateResidualRisk($riskId);
        
        return ['success' => true, 'control_id' => $controlId];
    }
    
    public function logIncident($riskId, $data) {
        $incidentId = Capsule::table('mod_risk_incidents')->insertGetId([
            'risk_id' => $riskId,
            'incident_date' => $data['date'] ?? Carbon::now(),
            'severity' => $data['severity'] ?? 'moderate',
            'description' => $data['description'],
            'financial_impact' => $data['financial_impact'] ?? 0,
            'response_status' => 'detected',
        ]);
        
        Capsule::table('mod_risk_assessments')
            ->where('risk_id', $riskId)
            ->update(['last_reviewed_at' => Carbon::now()]);
        
        return ['success' => true, 'incident_id' => $incidentId];
    }
    
    public function logRiskIndicator($category, $indicatorName, $value) {
        Capsule::table('mod_risk_metrics_log')->insert([
            'category' => $category,
            'indicator_name' => $indicatorName,
            'value' => $value,
            'recorded_at' => Carbon::now(),
        ]);
    }
    
    public function updateRiskIndicator($riskId, $indicatorName, $value) {
        $indicator = Capsule::table('mod_risk_indicators')
            ->where('risk_id', $riskId)
            ->where('indicator_name', $indicatorName)
            ->first();
        
        if ($indicator) {
            Capsule::table('mod_risk_indicators')
                ->where('id', $indicator->id)
                ->update([
                    'current_value' => $value,
                    'last_measured_at' => Carbon::now(),
                ]);
        }
    }
}
```

### lib/RiskCalculator.php

```php
<?php
/**
 * Risk Calculator
 */

namespace WHMCS\Module\RiskAssessment;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class RiskCalculator {
    
    public function calculateOverallRisk() {
        $categories = Capsule::table('mod_risk_categories')->where('is_active', 1)->get();
        
        $totalWeightedScore = 0;
        $totalWeight = 0;
        
        foreach ($categories as $category) {
            $categoryRisk = $this->calculateCategoryRisk($category->id);
            $totalWeightedScore += $categoryRisk * $category->weight;
            $totalWeight += $category->weight;
        }
        
        $overallScore = $totalWeight > 0 ? ($totalWeightedScore / $totalWeight) : 0;
        
        return [
            'overall_score' => round($overallScore, 2),
            'risk_level' => $this->determineRiskLevel($overallScore),
            'last_calculated' => Carbon::now()->toDateTimeString(),
        ];
    }
    
    protected function calculateCategoryRisk($categoryId) {
        $risks = Capsule::table('mod_risk_assessments')
            ->where('category_id', $categoryId)
            ->where('is_active', 1)
            ->whereIn('status', ['identified', 'assessed', 'mitigating', 'monitoring'])
            ->get();
        
        if ($risks->isEmpty()) {
            return 0;
        }
        
        // Aggregate risk using weighted average
        $totalScore = 0;
        $totalWeight = 0;
        
        foreach ($risks as $risk) {
            $weight = $this->getRiskWeight($risk->risk_level);
            $totalScore += $risk->risk_score * $weight;
            $totalWeight += $weight;
        }
        
        return $totalWeight > 0 ? ($totalScore / $totalWeight) : 0;
    }
    
    protected function getRiskWeight($level) {
        $weights = [
            'critical' => 4,
            'high' => 3,
            'medium' => 2,
            'low' => 1,
        ];
        return $weights[$level] ?? 1;
    }
    
    protected function determineRiskLevel($score) {
        if ($score >= 75) return 'critical';
        if ($score >= 50) return 'high';
        if ($score >= 25) return 'medium';
        return 'low';
    }
    
    public function getOverview() {
        $allRisks = Capsule::table('mod_risk_assessments')->where('is_active', 1)->get();
        
        return [
            'total_risks' => $allRisks->count(),
            'by_level' => [
                'critical' => $allRisks->where('risk_level', 'critical')->count(),
                'high' => $allRisks->where('risk_level', 'high')->count(),
                'medium' => $allRisks->where('risk_level', 'medium')->count(),
                'low' => $allRisks->where('risk_level', 'low')->count(),
            ],
            'by_status' => [
                'identified' => $allRisks->where('status', 'identified')->count(),
                'assessed' => $allRisks->where('status', 'assessed')->count(),
                'mitigating' => $allRisks->where('status', 'mitigating')->count(),
                'monitoring' => $allRisks->where('status', 'monitoring')->count(),
                'accepted' => $allRisks->where('status', 'accepted')->count(),
            ],
        ];
    }
    
    public function getRiskByCategory() {
        return Capsule::table('mod_risk_assessments as r')
            ->join('mod_risk_categories as c', 'r.category_id', '=', 'c.id')
            ->where('r.is_active', 1)
            ->groupBy('c.id')
            ->selectRaw('c.category_name, c.category_code, COUNT(*) as risk_count, AVG(r.risk_score) as avg_score')
            ->get();
    }
    
    public function getRiskByLevel() {
        return Capsule::table('mod_risk_assessments')
            ->where('is_active', 1)
            ->groupBy('risk_level')
            ->selectRaw('risk_level, COUNT(*) as count, AVG(risk_score) as avg_score, MAX(risk_score) as max_score')
            ->get();
    }
    
    public function calculateRiskTrend($days = 30) {
        $startDate = Carbon::now()->subDays($days)->toDateString();
        
        $historicalRisks = Capsule::table('mod_risk_assessments')
            ->whereNotNull('assessed_at')
            ->get()
            ->filter(function($risk) use ($startDate) {
                $assessedAt = Carbon::parse($risk->assessed_at)->toDateString();
                return $assessedAt >= $startDate;
            });
        
        return [
            'trend' => 'stable',
            'change_percentage' => 0,
            'data_points' => $historicalRisks->count(),
        ];
    }
}
```

### lib/RiskIndicatorMonitor.php

```php
<?php
/**
 * Risk Indicator Monitor
 */

namespace WHMCS\Module\RiskAssessment;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class RiskIndicatorMonitor {
    
    public function checkAllIndicators() {
        $indicators = Capsule::table('mod_risk_indicators')
            ->where('alert_enabled', 1)
            ->get();
        
        $alerts = [];
        
        foreach ($indicators as $indicator) {
            $status = $this->checkIndicator($indicator);
            if ($status) {
                $alerts[] = $status;
            }
        }
        
        return $alerts;
    }
    
    protected function checkIndicator($indicator) {
        $currentValue = $indicator->current_value;
        $riskId = $indicator->risk_id;
        
        // Update current value from data source
        $currentValue = $this->fetchIndicatorValue($riskId, $indicator->indicator_name);
        
        Capsule::table('mod_risk_indicators')
            ->where('id', $indicator->id)
            ->update([
                'current_value' => $currentValue,
                'last_measured_at' => Carbon::now(),
            ]);
        
        // Check thresholds
        $direction = $indicator->value_direction;
        
        if ($indicator->threshold_critical !== null) {
            $breached = $direction === 'below_better' 
                ? $currentValue <= $indicator->threshold_critical
                : $currentValue >= $indicator->threshold_critical;
            
            if ($breached) {
                return $this->createAlert($indicator, 'critical', $currentValue);
            }
        }
        
        if ($indicator->threshold_warning !== null) {
            $breached = $direction === 'below_better'
                ? $currentValue <= $indicator->threshold_warning
                : $currentValue >= $indicator->threshold_warning;
            
            if ($breached) {
                return $this->createAlert($indicator, 'warning', $currentValue);
            }
        }
        
        return null;
    }
    
    protected function fetchIndicatorValue($riskId, $indicatorName) {
        switch ($indicatorName) {
            case 'invoice_failure_rate':
                $total = Capsule::table('tblinvoices')
                    ->where('created_at', '>=', Carbon::now()->subDays(30))
                    ->count();
                $failed = Capsule::table('tblinvoices')
                    ->whereIn('status', ['Overdue', 'Unpaid'])
                    ->where('duedate', '<', Carbon::now()->toDateString())
                    ->count();
                return $total > 0 ? ($failed / $total) * 100 : 0;
                
            case 'service_availability':
                $total = Capsule::table('tblhosting')->count();
                $down = Capsule::table('tblhosting')->whereIn('domainstatus', ['Suspended', 'Terminated'])->count();
                return $total > 0 ? (($total - $down) / $total) * 100 : 100;
                
            case 'compliance_score':
                return Capsule::table('mod_compliance_results')
                    ->where('result', 'pass')
                    ->where('check_date', '>=', Carbon::now()->subDays(30))
                    ->count() * 10;
                
            default:
                return Capsule::table('mod_risk_metrics_log')
                    ->where('indicator_name', $indicatorName)
                    ->orderBy('recorded_at', 'desc')
                    ->value('value') ?? 0;
        }
    }
    
    protected function createAlert($indicator, $level, $currentValue) {
        $alertData = [
            'risk_id' => $indicator->risk_id,
            'indicator_name' => $indicator->indicator_name,
            'level' => $level,
            'current_value' => $currentValue,
            'threshold' => $level === 'critical' 
                ? $indicator->threshold_critical 
                : $indicator->threshold_warning,
            'timestamp' => Carbon::now()->toDateTimeString(),
        ];
        
        Capsule::table('mod_risk_alerts')->insert($alertData);
        
        return $alertData;
    }
    
    public function generateAlerts() {
        $alerts = Capsule::table('mod_risk_alerts')
            ->where('created_at', '>=', Carbon::now()->subHours(24))
            ->where('acknowledged', 0)
            ->get();
        
        if ($alerts->isEmpty()) {
            return;
        }
        
        $email = \App::get_config('risk_assessment')['alert_email'] ?? '';
        
        if ($email) {
            $this->sendAlertEmail($email, $alerts);
        }
    }
    
    protected function sendAlertEmail($email, $alerts) {
        $subject = 'Risk Assessment Alert - ' . count($alerts) . ' indicator(s) breached';
        $body = "Risk indicator alerts have been generated:\n\n";
        
        foreach ($alerts as $alert) {
            $body .= "- {$alert->indicator_name}: {$alert->level} (value: {$alert->current_value})\n";
        }
        
        sendEmail($subject, $email, $body);
    }
}
```

## API Endpoints

```
GET  /api/v1/risk/overview                 - Get overall risk score
GET  /api/v1/risk/dashboard               - Get risk dashboard
GET  /api/v1/risk/risks                   - Get all risks
GET  /api/v1/risk/risks/{risk_id}         - Get specific risk
POST /api/v1/risk/risks                   - Add new risk
PUT  /api/v1/risk/risks/{risk_id}          - Update risk
GET  /api/v1/risk/categories              - Get risk categories
GET  /api/v1/risk/incidents               - Get risk incidents
POST /api/v1/risk/incidents               - Log risk incident
GET  /api/v1/risk/indicators              - Get risk indicators
POST /api/v1/risk/indicators/check        - Check all indicators
```

## Hooks Integration

```php
// Update financial risk when payment fails
add_hook('InvoicePaymentFailed', 1, function($params) {
    $manager = new \WHMCS\Module\RiskAssessment\RiskManager();
    $manager->updateRiskIndicator('FIN', 'payment_failure_rate', $params['amount'] ?? 0);
});

// Track operational risk from service issues
add_hook('ServiceSuspended', 1, function($params) {
    $manager = new \WHMCS\Module\RiskAssessment\RiskManager();
    $manager->logRiskIndicator('OPS', 'suspension_count', 1);
    $manager->logIncident('OPS-001', [
        'severity' => 'moderate',
        'description' => 'Service suspended: ' . $params['service_id'],
    ]);
});

// Compliance risk updates
add_hook('ComplianceCheckComplete', 1, function($params) {
    if ($params['result'] === 'fail') {
        $manager = new \WHMCS\Module\RiskAssessment\RiskManager();
        $manager->logRiskIndicator('COMP', 'compliance_violation', 1);
    }
});
```
