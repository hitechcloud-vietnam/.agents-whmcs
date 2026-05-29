# WHMCS SaaS Metrics DevKit

## Overview

A comprehensive metrics tracking and analytics system for WHMCS SaaS operations, providing real-time usage tracking, MRR/ARR calculations, churn analysis, cohort tracking, and custom metric definitions.

## Features

- MRR/ARR tracking
- Usage metrics
- Customer health scoring
- Churn analysis
- Cohort analysis
- Revenue analytics
- Custom KPIs
- Real-time dashboards
- Metric alerts
- Report generation

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_saas_metrics` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `metric_name` VARCHAR(100) NOT NULL,
    `metric_code` VARCHAR(50) NOT NULL,
    `metric_type` ENUM('revenue', 'usage', 'engagement', 'churn', 'custom') NOT NULL,
    `aggregation` ENUM('sum', 'avg', 'count', 'min', 'max') NOT NULL DEFAULT 'sum',
    `unit` VARCHAR(50) NULL,
    `description` TEXT NULL,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_metric_code` (`metric_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_saas_metric_values` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `metric_id` INT UNSIGNED NOT NULL,
    `entity_type` VARCHAR(50) NULL,
    `entity_id` INT UNSIGNED NULL,
    `value` DECIMAL(15,4) NOT NULL,
    `recorded_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_metric_date` (`metric_id`, `recorded_at`),
    CONSTRAINT `fk_metric_value` FOREIGN KEY (`metric_id`) REFERENCES `mod_saas_metrics`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_saas_revenue_snapshots` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `snapshot_date` DATE NOT NULL,
    `mrr` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    `arr` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    `arr` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    `new_mrr` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    `expansion_mrr` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    `churned_mrr` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    `contraction_mrr` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    `net_new_mrr` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    `active_customers` INT UNSIGNED NOT NULL DEFAULT 0,
    `active_subscriptions` INT UNSIGNED NOT NULL DEFAULT 0,
    `arpu` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_snapshot_date` (`snapshot_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_saas_customer_health` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `user_id` INT UNSIGNED NOT NULL,
    `health_score` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    `engagement_score` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    `usage_score` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    `payment_score` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    `support_score` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    `churn_risk` ENUM('low', 'medium', 'high', 'critical') NOT NULL DEFAULT 'low',
    `last_activity_at` DATETIME NULL,
    `risk_factors` JSON NULL,
    `calculated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_user_health` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Class

```php
<?php
/**
 * WHMCS SaaS Metrics Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/SaasMetricsManager.php';
require_once __DIR__ . '/lib/RevenueCalculator.php';
require_once __DIR__ . '/lib/ChurnAnalyzer.php';

function whmcs_saas_metrics_activate() {
    $manager = new SaasMetricsManager();
    return $manager->activate();
}

function whmcs_saas_metrics_deactivate() {
    return ['success' => true, 'msg' => 'SaaS Metrics module deactivated'];
}

function whmcs_saas_metrics_config() {
    return [
        'currency' => [
            'FriendlyName' => 'Revenue Currency',
            'Type' => 'text',
            'Default' => 'USD',
        ],
        'billing_period' => [
            'FriendlyName' => 'Billing Period',
            'Type' => 'dropdown',
            'Options' => [
                'monthly' => 'Monthly',
                'quarterly' => 'Quarterly',
                'annually' => 'Annually',
            ],
            'Default' => 'monthly',
        ],
    ];
}

function whmcs_saas_metrics_get_overview() {
    $manager = new SaasMetricsManager();
    return $manager->getOverview();
}

function whmcs_saas_metrics_get_mrr() {
    $calc = new RevenueCalculator();
    return $calc->calculateMRR();
}

function whmcs_saas_metrics_get_arr() {
    $calc = new RevenueCalculator();
    return $calc->calculateARR();
}

function whmcs_saas_metrics_get_customer_health($userId) {
    $manager = new SaasMetricsManager();
    return $manager->getCustomerHealth($userId);
}

function whmcs_saas_metrics_record_usage($userId, $metricCode, $value) {
    $manager = new SaasMetricsManager();
    return $manager->recordUsage($userId, $metricCode, $value);
}

add_hook('DailyCronJob', 1, function() {
    $calc = new RevenueCalculator();
    $calc->calculateDailySnapshot();
    
    $manager = new SaasMetricsManager();
    $manager->updateAllCustomerHealth();
});

add_hook('InvoicePayment', 1, function($params) {
    $calc = new RevenueCalculator();
    $calc->recordPayment($params);
});
```

### lib/SaasMetricsManager.php

```php
<?php
namespace WHMCS\Module\SaasMetrics;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class SaasMetricsManager {
    
    public function activate() {
        try {
            $this->createTables();
            $this->initializeDefaultMetrics();
            return ['success' => true, 'msg' => 'SaaS Metrics module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_saas_metrics` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `metric_name` VARCHAR(100) NOT NULL,
                `metric_code` VARCHAR(50) NOT NULL,
                `metric_type` VARCHAR(20) NOT NULL,
                `aggregation` VARCHAR(20) NOT NULL DEFAULT 'sum',
                `unit` VARCHAR(50) NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_saas_metric_values` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `metric_id` INT UNSIGNED NOT NULL,
                `entity_type` VARCHAR(50) NULL,
                `entity_id` INT UNSIGNED NULL,
                `value` DECIMAL(15,4) NOT NULL,
                `recorded_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_saas_revenue_snapshots` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `snapshot_date` DATE NOT NULL UNIQUE,
                `mrr` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
                `arr` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
                `new_mrr` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
                `churned_mrr` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
                `active_customers` INT UNSIGNED NOT NULL DEFAULT 0,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_saas_customer_health` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL UNIQUE,
                `health_score` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                `engagement_score` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                `usage_score` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                `payment_score` DECIMAL(5,2) NOT NULL DEFAULT 100.00,
                `churn_risk` VARCHAR(20) NOT NULL DEFAULT 'low',
                `calculated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function initializeDefaultMetrics() {
        $metrics = [
            ['name' => 'Monthly Recurring Revenue', 'code' => 'MRR', 'type' => 'revenue'],
            ['name' => 'Annual Recurring Revenue', 'code' => 'ARR', 'type' => 'revenue'],
            ['name' => 'Active Users', 'code' => 'ACTIVE_USERS', 'type' => 'usage'],
            ['name' => 'API Calls', 'code' => 'API_CALLS', 'type' => 'usage'],
            ['name' => 'Storage Used', 'code' => 'STORAGE_USED', 'type' => 'usage'],
            ['name' => 'Support Tickets', 'code' => 'SUPPORT_TICKETS', 'type' => 'engagement'],
        ];
        
        foreach ($metrics as $metric) {
            if (!Capsule::table('mod_saas_metrics')->where('metric_code', $metric['code'])->exists()) {
                Capsule::table('mod_saas_metrics')->insert($metric);
            }
        }
    }
    
    public function getOverview() {
        $latestSnapshot = Capsule::table('mod_saas_revenue_snapshots')
            ->orderBy('snapshot_date', 'desc')
            ->first();
        
        $previousSnapshot = Capsule::table('mod_saas_revenue_snapshots')
            ->where('snapshot_date', '<', $latestSnapshot->snapshot_date ?? Carbon::today())
            ->orderBy('snapshot_date', 'desc')
            ->first();
        
        return [
            'mrr' => $latestSnapshot->mrr ?? 0,
            'mrr_change' => $latestSnapshot && $previousSnapshot 
                ? (($latestSnapshot->mrr - $previousSnapshot->mrr) / $previousSnapshot->mrr) * 100 
                : 0,
            'arr' => $latestSnapshot->arr ?? 0,
            'active_customers' => $latestSnapshot->active_customers ?? 0,
            'arpu' => $latestSnapshot->arpu ?? 0,
            'churn_rate' => $this->calculateChurnRate(),
        ];
    }
    
    protected function calculateChurnRate() {
        $currentMonth = Capsule::table('mod_saas_revenue_snapshots')
            ->where('snapshot_date', '>=', Carbon::now()->startOfMonth()->toDateString())
            ->first();
        
        $previousMonth = Capsule::table('mod_saas_revenue_snapshots')
            ->where('snapshot_date', '>=', Carbon::now()->subMonth()->startOfMonth()->toDateString())
            ->where('snapshot_date', '<', Carbon::now()->startOfMonth()->toDateString())
            ->first();
        
        if (!$currentMonth || !$previousMonth || $previousMonth->mrr == 0) {
            return 0;
        }
        
        return ($currentMonth->churned_mrr / $previousMonth->mrr) * 100;
    }
    
    public function getCustomerHealth($userId) {
        $health = Capsule::table('mod_saas_customer_health')->where('user_id', $userId)->first();
        
        if (!$health) {
            return $this->calculateCustomerHealth($userId);
        }
        
        return $health;
    }
    
    public function calculateCustomerHealth($userId) {
        $client = Capsule::table('tblclients')->where('id', $userId)->first();
        
        // Payment score
        $paidInvoices = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->where('status', 'Paid')
            ->count();
        $totalInvoices = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->count();
        $paymentScore = $totalInvoices > 0 ? ($paidInvoices / $totalInvoices) * 100 : 100;
        
        // Engagement score (ticket activity, login frequency)
        $ticketsLast30Days = Capsule::table('tbltickets')
            ->where('userid', $userId)
            ->where('created_at', '>=', Carbon::now()->subDays(30))
            ->count();
        $engagementScore = min(100, $ticketsLast30Days * 10);
        
        // Usage score
        $usageMetrics = Capsule::table('mod_saas_metric_values')
            ->join('mod_saas_metrics', 'mod_saas_metric_values.metric_id', '=', 'mod_saas_metrics.id')
            ->where('mod_saas_metric_values.entity_id', $userId)
            ->where('mod_saas_metrics.metric_type', 'usage')
            ->where('mod_saas_metric_values.recorded_at', '>=', Carbon::now()->subDays(30))
            ->selectRaw('SUM(mod_saas_metric_values.value) as total_usage')
            ->first();
        $usageScore = min(100, ($usageMetrics->total_usage ?? 0) / 10);
        
        // Overall health score
        $healthScore = ($paymentScore * 0.4) + ($engagementScore * 0.3) + ($usageScore * 0.3);
        
        $churnRisk = $healthScore >= 70 ? 'low' : ($healthScore >= 40 ? 'medium' : ($healthScore >= 20 ? 'high' : 'critical'));
        
        Capsule::table('mod_saas_customer_health')->updateOrInsert(
            ['user_id' => $userId],
            [
                'health_score' => $healthScore,
                'engagement_score' => $engagementScore,
                'usage_score' => $usageScore,
                'payment_score' => $paymentScore,
                'churn_risk' => $churnRisk,
                'last_activity_at' => Carbon::now(),
                'calculated_at' => Carbon::now(),
            ]
        );
        
        return [
            'user_id' => $userId,
            'health_score' => $healthScore,
            'payment_score' => $paymentScore,
            'engagement_score' => $engagementScore,
            'usage_score' => $usageScore,
            'churn_risk' => $churnRisk,
        ];
    }
    
    public function recordUsage($userId, $metricCode, $value) \{\}
    
    public function updateAllCustomerHealth() {
        $users = Capsule::table('tblclients')->get();
        foreach ($users as $user) {
            $this->calculateCustomerHealth($user->id);
        }
    }
}
```

### lib/RevenueCalculator.php

```php
<?php
namespace WHMCS\Module\SaasMetrics;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class RevenueCalculator {
    
    public function calculateMRR() {
        $activeServices = Capsule::table('tblhosting')
            ->whereIn('domainstatus', ['Active', 'Pending'])
            ->get();
        
        $mrr = 0;
        foreach ($activeServices as $service) {
            $billingCycle = $service->billingcycle ?? 'Monthly';
            $amount = $service->amount ?? 0;
            
            switch ($billingCycle) {
                case 'Monthly':
                    $mrr += $amount;
                    break;
                case 'Quarterly':
                    $mrr += $amount / 3;
                    break;
                case 'Annually':
                    $mrr += $amount / 12;
                    break;
                case 'Biennially':
                    $mrr += $amount / 24;
                    break;
                case 'Triennially':
                    $mrr += $amount / 36;
                    break;
            }
        }
        
        return [
            'mrr' => round($mrr, 2),
            'calculated_at' => Carbon::now()->toDateTimeString(),
        ];
    }
    
    public function calculateARR() {
        $mrrData = $this->calculateMRR();
        return [
            'arr' => $mrrData['mrr'] * 12,
            'mrr' => $mrrData['mrr'],
        ];
    }
    
    public function calculateDailySnapshot() {
        $today = Carbon::today()->toDateString();
        
        $mrr = $this->calculateMRR();
        
        $activeCustomers = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->distinct('userid')
            ->count('userid');
        
        $newMRR = $this->calculateNewMRR();
        $churnedMRR = $this->calculateChurnedMRR();
        
        Capsule::table('mod_saas_revenue_snapshots')->updateOrInsert(
            ['snapshot_date' => $today],
            [
                'mrr' => $mrr['mrr'],
                'arr' => $mrr['mrr'] * 12,
                'new_mrr' => $newMRR,
                'churned_mrr' => $churnedMRR,
                'active_customers' => $activeCustomers,
                'arpu' => $activeCustomers > 0 ? $mrr['mrr'] / $activeCustomers : 0,
            ]
        );
        
        return ['success' => true];
    }
    
    protected function calculateNewMRR() {
        $monthStart = Carbon::now()->startOfMonth()->toDateString();
        
        $newServices = Capsule::table('tblhosting')
            ->where('createddate', '>=', $monthStart)
            ->where('domainstatus', 'Active')
            ->get();
        
        $newMRR = 0;
        foreach ($newServices as $service) {
            $newMRR += $service->amount ?? 0;
        }
        
        return $newMRR;
    }
    
    protected function calculateChurnedMRR() {
        $monthStart = Carbon::now()->startOfMonth()->toDateString();
        
        $churnedServices = Capsule::table('tblhosting')
            ->where('domainstatus', 'Terminated')
            ->get();
        
        $churnedMRR = 0;
        foreach ($churnedServices as $service) {
            $churnedMRR += $service->amount ?? 0;
        }
        
        return $churnedMRR;
    }
    
    public function recordPayment($params) {
        $invoice = Capsule::table('tblinvoices')->where('id', $params['invoice_id'])->first();
        
        if (!$invoice) return;
        
        // Track when customer upgrades or adds services
        $metric = Capsule::table('mod_saas_metrics')->where('metric_code', 'REVENUE_EVENTS')->first();
        
        if ($metric) {
            Capsule::table('mod_saas_metric_values')->insert([
                'metric_id' => $metric->id,
                'entity_type' => 'user',
]            'entity_id' => $params['user_id'],
                'value' => $params['amount'],
                'recorded_at' => Carbon::now(),
            ]);
        }
    }
}
```

### lib/ChurnAnalyzer.php

```php
<?php
namespace WHMCS\Module\SaasMetrics;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ChurnAnalyzer {
    
    public function getChurnRate($period = 'monthly') {
        $startDate = $this->getPeriodStart($period);
        
        $churnedCustomers = Capsule::table('tblhosting')
            ->whereIn('domainstatus', ['Terminated', 'Cancelled'])
            ->where('terminateddate', '>=', $startDate)
            ->distinct('userid')
            ->count('userid');
        
        $totalCustomers = Capsule::table('tblhosting')
            ->where('createddate', '<=', $startDate)
            ->distinct('userid')
            ->count('userid');
        
        $churnRate = $totalCustomers > 0 ? ($churnedCustomers / $totalCustomers) * 100 : 0;
        
        return [
            'period' => $period,
            'churned_customers' => $churnedCustomers,
            'total_customers_at_start' => $totalCustomers,
            'churn_rate' => round($churnRate, 2),
        ];
    }
    
    protected function getPeriodStart($period) {
        switch ($period) {
            case 'weekly':
                return Carbon::now()->subWeek()->toDateString();
            case 'monthly':
                return Carbon::now()->subMonth()->toDateString();
            case 'quarterly':
                return Carbon::now()->subMonths(3)->toDateString();
            case 'yearly':
                return Carbon::now()->subYear()->toDateString();
            default:
                return Carbon::now()->subMonth()->toDateString();
        }
    }
    
    public function getHighChurnRiskCustomers($limit = 10) {
        return Capsule::table('mod_saas_customer_health')
            ->whereIn('churn_risk', ['high', 'critical'])
            ->orderBy('health_score', 'asc')
            ->limit($limit)
            ->get();
    }
    
    public function getCohortAnalysis($cohortPeriod = 'monthly') {
        $cohorts = [];
        
        for ($i = 0; $i < 6; $i++) {
            $cohortMonth = Carbon::now()->subMonths($i)->startOfMonth();
            $cohortEnd = $cohortMonth->copy()->endOfMonth();
            
            $cohortUsers = Capsule::table('tblclients')
                ->where('datecreated', '>=', $cohortMonth->toDateString())
                ->where('datecreated', '<=', $cohortEnd->toDateString())
                ->pluck('id')
                ->toArray();
            
            $retainedCount = count($cohortUsers);
            $retentionData = [];
            
            for ($month = 0; $month <= $i; $month++) {
                $checkDate = $cohortMonth->copy()->addMonths($month);
                $retained = Capsule::table('tblhosting')
                    ->whereIn('userid', $cohortUsers)
                    ->where('domainstatus', 'Active')
                    ->count();
                
                $retentionData[] = [
                    'month' => $month,
                    'retained' => $retained,
                    'retention_rate' => count($cohortUsers) > 0 ? ($retained / count($cohortUsers)) * 100 : 0,
                ];
            }
            
            $cohorts[] = [
                'cohort_month' => $cohortMonth->toDateString(),
                'cohort_size' => count($cohortUsers),
                'retention' => $retentionData,
            ];
        }
        
        return $cohorts;
    }
}
```

## API Endpoints

```
GET  /api/v1/saas/overview              - Get metrics overview
GET  /api/v1/saas/mrr                   - Get MRR
GET  /api/v1/saas/arr                   - Get ARR
GET  /api/v1/saas/revenue-snapshots     - Get revenue snapshots
GET  /api/v1/saas/customer-health/{id}  - Get customer health
GET  /api/v1/saas/churn-rate            - Get churn rate
GET  /a pi/v1/saas/cohorts              - Get cohort analysis
POST /api/v1/saas/usage                - Record usage
GET  /api/v1/saas/metrics/{code}         - Get specific metric
```
