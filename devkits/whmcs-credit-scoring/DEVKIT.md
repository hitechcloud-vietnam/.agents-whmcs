# WHMCS Credit Scoring DevKit

## Overview

A comprehensive credit scoring and risk assessment system for WHMCS that evaluates customer creditworthiness, manages credit limits, tracks payment behavior, and provides credit recommendations based on multiple factors.

## Features

- Multi-factor credit scoring algorithm
- Credit limit management
- Payment behavior analysis
- Credit risk categories
- Automated credit decisions
- Credit history tracking
- Credit limit adjustments
- Fraud risk scoring
- Late payment prediction
- Credit recommendations

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_credit_profiles` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `user_id` INT UNSIGNED NOT NULL,
    `credit_score` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    `credit_limit` DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    `available_credit` DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    `credit_category` ENUM('excellent', 'good', 'fair', 'poor', 'no_history') NOT NULL DEFAULT 'no_history',
    `risk_level` ENUM('low', 'medium', 'high', 'critical') NOT NULL DEFAULT 'medium',
    `payment_reliability` DECIMAL(5,2) NOT NULL DEFAULT 100.00,
    `credit_utilization` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    `last_credit_check` DATETIME NULL,
    `credit_review_date` DATE NULL,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_user_profile` (`user_id`),
    CONSTRAINT `fk_profile_user` FOREIGN KEY (`user_id`) REFERENCES `tblusers`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_credit_factors` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `user_id` INT UNSIGNED NOT NULL,
    `factor_type` VARCHAR(50) NOT NULL,
    `factor_weight` DECIMAL(3,2) NOT NULL DEFAULT 1.00,
    `factor_score` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    `contributing_factors` JSON NULL,
    `evidence_data` JSON NULL,
    `calculated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_user_factors` (`user_id`),
    CONSTRAINT `fk_factor_user` FOREIGN KEY (`user_id`) REFERENCES `tblusers`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_credit_history` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `user_id` INT UNSIGNED NOT NULL,
    `event_type` ENUM('inquiry', 'limit_increase', 'limit_decrease', 'score_change', 'credit_issued', 'default', 'recovery') NOT NULL,
    `previous_value` DECIMAL(10,2) NULL,
    `new_value` DECIMAL(10,2) NULL,
    `credit_limit_change` DECIMAL(12,2) NULL,
    `trigger_reason` TEXT NULL,
    `approved_by` INT UNSIGNED NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_user_history` (`user_id`),
    INDEX `idx_created_at` (`created_at`),
    CONSTRAINT `fk_history_user` FOREIGN KEY (`user_id`) REFERENCES `tblusers`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_credit_decisions` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `user_id` INT UNSIGNED NOT NULL,
    `decision_type` ENUM('new_credit', 'limit_increase', 'limit_decrease', 'service_approval', 'payment_plan', 'suspension') NOT NULL,
    `requested_amount` DECIMAL(12,2) NULL,
    `decision` ENUM('approved', 'denied', 'pending', 'manual_review') NOT NULL,
    `decision_score` DECIMAL(5,2) NULL,
    `risk_assessment` TEXT NULL,
    `conditions` JSON NULL,
    `expires_at` DATETIME NULL,
    `reviewed_by` INT UNSIGNED NULL,
    `reviewed_at` DATETIME NULL,
    `notes` TEXT NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_user_decisions` (`user_id`),
    INDEX `idx_decision_date` (`created_at`),
    CONSTRAINT `fk_decision_user` FOREIGN KEY (`user_id`) REFERENCES `tblusers`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Files

### credit_scoring.php

```php
<?php
/**
 * WHMCS Credit Scoring Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/CreditScorer.php';
require_once __DIR__ . '/lib/CreditLimitManager.php';
require_once __DIR__ . '/lib/CreditDecisionEngine.php';

use WHMCS\Module\CreditScoring\CreditScorer;
use WHMCS\Module\CreditScoring\CreditLimitManager;
use WHMCS\Module\CreditScoring\CreditDecisionEngine;

function whmcs_credit_scoring_activate() {
    $scorer = new CreditScorer();
    return $scorer->activate();
}

function whmcs_credit_scoring_deactivate() {
    return ['success' => true, 'msg' => 'Credit Scoring module deactivated'];
}

function whmcs_credit_scoring_config() {
    return [
        'default_limit' => [
            'FriendlyName' => 'Default Credit Limit',
            'Type' => 'text',
            'Size' => '15',
            'Default' => '1000',
        ],
        'min_score_threshold' => [
            'FriendlyName' => 'Minimum Score Threshold',
            'Type' => 'dropdown',
            'Options' => [
                '300' => '300 (Poor)',
                '500' => '500 (Fair)',
                '600' => '600 (Good)',
                '700' => '700 (Excellent)',
            ],
            'Default' => '500',
        ],
        'auto_approve_limit' => [
            'FriendlyName' => 'Auto-approve Limit',
            'Type' => 'text',
            'Size' => '15',
            'Default' => '500',
        ],
        'score_weight_payment' => [
            'FriendlyName' => 'Payment History Weight',
            'Type' => 'dropdown',
            'Options' => [
                '35' => '35%',
                '40' => '40%',
                '45' => '45%',
            ],
            'Default' => '35',
        ],
        'score_weight_utilization' => [
            'FriendlyName' => 'Credit Utilization Weight',
            'Type' => 'dropdown',
            'Options' => [
                '30' => '30%',
                '25' => '25%',
                '20' => '20%',
            ],
            'Default' => '30',
        ],
    ];
}

/**
 * Calculate credit score for user
 */
function whmcs_credit_scoring_calculate_score($userId) {
    $scorer = new CreditScorer();
    return $scorer->calculateScore($userId);
}

/**
 * Get credit profile
 */
function whmcs_credit_scoring_get_profile($userId) {
    $scorer = new CreditScorer();
    return $scorer->getProfile($userId);
}

/**
 * Get credit limit recommendation
 */
function whmcs_credit_scoring_get_recommended_limit($userId) {
    $limitManager = new CreditLimitManager();
    return $limitManager->getRecommendedLimit($userId);
}

/**
 * Make credit decision
 */
function whmcs_credit_scoring_make_decision($userId, $type, $amount = null) {
    $engine = new CreditDecisionEngine();
    return $engine->makeDecision($userId, $type, $amount);
}

/**
 * Update credit limit
 */
function whmcs_credit_scoring_update_limit($userId, $newLimit, $reason = null) {
    $limitManager = new CreditLimitManager();
    return $limitManager->updateLimit($userId, $newLimit, $reason);
}

// Hooks
add_hook('DailyCronJob', 1, function() {
    $scorer = new CreditScorer();
    $scorer->updateAllScores();
    $scorer->identifyRiskAccounts();
});

add_hook('InvoicePayment', 1, function($params) {
    $scorer = new CreditScorer();
    $scorer->recordPaymentBehavior($params['user_id'], $params['invoice_id']);
});

add_hook('InvoiceCreated', 1, function($params) {
    $scorer = new CreditScorer();
    $scorer->checkOverdueInvoices($params['user_id']);
});
```

### lib/CreditScorer.php

```php
<?php
/**
 * Credit Scorer
 */

namespace WHMCS\Module\CreditScoring;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class CreditScorer {
    
    protected $factorWeights = [
        'payment_history' => 0.35,
        'credit_utilization' => 0.30,
        'account_age' => 0.15,
        'payment_frequency' => 0.10,
        'account_activity' => 0.10,
    ];
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Credit Scoring module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_credit_profiles` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL UNIQUE,
                `credit_score` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                `credit_limit` DECIMAL(12,2) NOT NULL DEFAULT 0.00,
                `available_credit` DECIMAL(12,2) NOT NULL DEFAULT 0.00,
                `credit_category` ENUM('excellent', 'good', 'fair', 'poor', 'no_history') NOT NULL DEFAULT 'no_history',
                `risk_level` ENUM('low', 'medium', 'high', 'critical') NOT NULL DEFAULT 'medium',
                `payment_reliability` DECIMAL(5,2) NOT NULL DEFAULT 100.00,
                `credit_utilization` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                `last_credit_check` DATETIME NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_credit_factors` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL,
                `factor_type` VARCHAR(50) NOT NULL,
                `factor_weight` DECIMAL(3,2) NOT NULL DEFAULT 1.00,
                `factor_score` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                `evidence_data` JSON NULL,
                `calculated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_credit_history` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL,
                `event_type` VARCHAR(50) NOT NULL,
                `previous_value` DECIMAL(10,2) NULL,
                `new_value` DECIMAL(10,2) NULL,
                `credit_limit_change` DECIMAL(12,2) NULL,
                `trigger_reason` TEXT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_credit_decisions` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL,
                `decision_type` VARCHAR(50) NOT NULL,
                `requested_amount` DECIMAL(12,2) NULL,
                `decision` ENUM('approved', 'denied', 'pending', 'manual_review') NOT NULL,
                `decision_score` DECIMAL(5,2) NULL,
                `risk_assessment` TEXT NULL,
                `reviewed_at` DATETIME NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function calculateScore($userId) {
        $factors = $this->calculateAllFactors($userId);
        $totalScore = 0;
        
        foreach ($this->factorWeights as $factorType => $weight) {
            if (isset($factors[$factorType])) {
                $totalScore += $factors[$factorType] * $weight;
            }
        }
        
        $totalScore = min(850, max(300, $totalScore));
        $category = $this->getScoreCategory($totalScore);
        $riskLevel = $this->getRiskLevel($totalScore);
        
        // Update profile
        $profile = $this->getOrCreateProfile($userId);
        Capsule::table('mod_credit_profiles')
            ->where('user_id', $userId)
            ->update([
                'credit_score' => $totalScore,
                'credit_category' => $category,
                'risk_level' => $riskLevel,
                'last_credit_check' => Carbon::now(),
                'payment_reliability' => $factors['payment_history'] ?? 0,
                'credit_utilization' => $factors['credit_utilization'] ?? 0,
            ]);
        
        // Store factors
        $this->storeFactors($userId, $factors);
        
        return [
            'user_id' => $userId,
            'credit_score' => $totalScore,
            'category' => $category,
            'risk_level' => $riskLevel,
            'factors' => $factors,
            'calculated_at' => Carbon::now()->toDateTimeString(),
        ];
    }
    
    protected function calculateAllFactors($userId) {
        return [
            'payment_history' => $this->calculatePaymentHistoryScore($userId),
            'credit_utilization' => $this->calculateUtilizationScore($userId),
            'account_age' => $this->calculateAccountAgeScore($userId),
            'payment_frequency' => $this->calculatePaymentFrequencyScore($userId),
            'account_activity' => $this->calculateActivityScore($userId),
        ];
    }
    
    protected function calculatePaymentHistoryScore($userId) {
        $totalInvoices = Capsule::table('tblinvoices')->where('userid', $userId)->count();
        if ($totalInvoices === 0) return 500;
        
        $paidInvoices = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->where('status', 'Paid')
            ->count();
        
        $onTimePayments = Capsule::table('tblinvoices as inv')
            ->join('tblaccounts as acc', 'inv.id', '=', 'acc.invoiceid')
            ->where('inv.userid', $userId)
            ->where('inv.status', 'Paid')
            ->whereRaw('acc.date <= inv.duedate')
            ->count();
        
        $paymentRate = ($totalInvoices > 0) ? ($paidInvoices / $totalInvoices) * 100 : 0;
        $onTimeRate = ($paidInvoices > 0) ? ($onTimePayments / $paidInvoices) * 100 : 0;
        
        $lateCount = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->whereIn('status', ['Overdue', 'Unpaid'])
            ->where('duedate', '<', Carbon::now()->toDateString())
            ->count();
        
        // Calculate score (base 300, max 850)
        $score = 300 + ($paymentRate * 3) + ($onTimeRate * 2);
        $score -= min($lateCount * 20, 200);
        
        return min(850, max(300, $score));
    }
    
    protected function calculateUtilizationScore($userId) {
        $profile = Capsule::table('mod_credit_profiles')->where('user_id', $userId)->first();
        
        if (!$profile || $profile->credit_limit <= 0) {
            return 500; // No credit history
        }
        
        $outstandingBalance = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->whereIn('status', ['Unpaid', 'Overdue'])
            ->sum('total');
        
        $utilization = ($profile->credit_limit > 0) 
            ? ($outstandingBalance / $profile->credit_limit) * 100 
            : 0;
        
        if ($utilization <= 30) return 700;
        if ($utilization <= 50) return 600;
        if ($utilization <= 75) return 500;
        if ($utilization <= 90) return 400;
        return 300;
    }
    
    protected function calculateAccountAgeScore($userId) {
        $client = Capsule::table('tblclients')->where('id', $userId)->first();
        
        if (!$client) return 300;
        
        $accountAge = Carbon::parse($client->datecreated)->diffInMonths(Carbon::now());
        
        if ($accountAge >= 36) return 700;
        if ($accountAge >= 24) return 600;
        if ($accountAge >= 12) return 500;
        if ($accountAge >= 6) return 400;
        if ($accountAge >= 3) return 350;
        return 300;
    }
    
    protected function calculatePaymentFrequencyScore($userId) {
        $invoiceCount = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->where('created_at', '>=', Carbon::now()->subMonths(6))
            ->count();
        
        if ($invoiceCount >= 12) return 700;
        if ($invoiceCount >= 8) return 600;
        if ($invoiceCount >= 4) return 500;
        if ($invoiceCount >= 1) return 400;
        return 300;
    }
    
    protected function calculateActivityScore($userId) {
        $recentActivity = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->where('created_at', '>=', Carbon::now()->subDays(90))
            ->count();
        
        if ($recentActivity >= 6) return 700;
        if ($recentActivity >= 4) return 600;
        if ($recentActivity >= 2) return 500;
        if ($recentActivity >= 1) return 400;
        return 300;
    }
    
    protected function getScoreCategory($score) {
        if ($score >= 750) return 'excellent';
        if ($score >= 650) return 'good';
        if ($score >= 500) return 'fair';
        return 'poor';
    }
    
    protected function getRiskLevel($score) {
        if ($score >= 700) return 'low';
        if ($score >= 500) return 'medium';
        if ($score >= 300) return 'high';
        return 'critical';
    }
    
    protected function getOrCreateProfile($userId) {
        $profile = Capsule::table('mod_credit_profiles')->where('user_id', $userId)->first();
        
        if (!$profile) {
            $defaultLimit = \App::get_config('credit_scoring')['default_limit'] ?? 1000;
            Capsule::table('mod_credit_profiles')->insert([
                'user_id' => $userId,
                'credit_limit' => $defaultLimit,
                'available_credit' => $defaultLimit,
            ]);
            return Capsule::table('mod_credit_profiles')->where('user_id', $userId)->first();
        }
        
        return $profile;
    }
    
    protected function storeFactors($userId, $factors) {
        foreach ($factors as $type => $score) {
            Capsule::table('mod_credit_factors')->insert([
                'user_id' => $userId,
                'factor_type' => $type,
                'factor_weight' => $this->factorWeights[$type] ?? 1.0,
                'factor_score' => $score,
                'calculated_at' => Carbon::now(),
            ]);
        }
    }
    
    public function getProfile($userId) {
        $profile = Capsule::table('mod_credit_profiles')->where('user_id', $userId)->first();
        
        if (!$profile) {
            return null;
        }
        
        $recentFactors = Capsule::table('mod_credit_factors')
            ->where('user_id', $userId)
            ->where('calculated_at', '>=', Carbon::now()->subHours(1))
            ->get()
            ->keyBy('factor_type');
        
        return [
            'user_id' => $userId,
            'credit_score' => $profile->credit_score,
            'credit_limit' => $profile->credit_limit,
            'available_credit' => $profile->available_credit,
            'category' => $profile->credit_category,
            'risk_level' => $profile->risk_level,
            'payment_reliability' => $profile->payment_reliability,
            'credit_utilization' => $profile->credit_utilization,
            'factors' => $recentFactors,
            'last_updated' => $profile->updated_at,
        ];
    }
    
    public function updateAllScores() {
        // Update scores monthly for all active users
        $users = Capsule::table('mod_credit_profiles')
            ->where('is_active', 1)
            ->where('last_credit_check', '<', Carbon::now()->subDays(30))
            ->pluck('user_id');
        
        foreach ($users as $userId) {
            $this->calculateScore($userId);
        }
    }
    
    public function recordPaymentBehavior($userId, $invoiceId) {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
        
        if (!$invoice || $invoice->userid != $userId) return;
        
        $onTime = Carbon::parse($invoice->duedate)->gte(Carbon::parse($invoice->datepaid ?? $invoice->duedate));
        
        Capsule::table('mod_credit_history')->insert([
            'user_id' => $userId,
            'event_type' => 'payment',
            'new_value' => $invoice->total,
            'trigger_reason' => $onTime ? 'On-time payment' : 'Late payment',
        ]);
    }
    
    public function checkOverdueInvoices($userId) {
        $overdueCount = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->where('status', 'Overdue')
            ->count();
        
        if ($overdueCount > 0) {
            Capsule::table('mod_credit_history')->insert([
                'user_id' => $userId,
                'event_type' => 'overdue_invoice',
                'new_value' => $overdueCount,
                'trigger_reason' => "$overdueCount overdue invoice(s)",
            ]);
        }
    }
    
    public function identifyRiskAccounts() {
        $profiles = Capsule::table('mod_credit_profiles')
            ->where('is_active', 1)
            ->where('credit_score', '<', 400)
            ->get();
        
        foreach ($profiles as $profile) {
            Capsule::table('mod_credit_history')->insert([
                'user_id' => $profile->user_id,
                'event_type' => 'high_risk_alert',
                'trigger_reason' => "Credit score dropped below 400: {$profile->credit_score}",
            ]);
        }
    }
}
```

### lib/CreditLimitManager.php

```php
<?php
/**
 * Credit Limit Manager
 */

namespace WHMCS\Module\CreditScoring;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class CreditLimitManager {
    
    protected $scorer;
    
    public function __construct() {
        $this->scorer = new CreditScorer();
    }
    
    public function getRecommendedLimit($userId) {
        $profile = Capsule::table('mod_credit_profiles')->where('user_id', $userId)->first();
        $score = $profile ? $profile->credit_score : 500;
        
        // Base limits by score
        $baseLimits = [
            'excellent' => 5000,
            'good' => 2500,
            'fair' => 1000,
            'poor' => 500,
            'no_history' => 250,
        ];
        
        $category = $this->scorer->getScoreCategory($score);
        $baseLimit = $baseLimits[$category] ?? 500;
        
        // Adjust for account history
        $client = Capsule::table('tblclients')->where('id', $userId)->first();
        if ($client) {
            $accountAge = Carbon::parse($client->datecreated)->diffInMonths(Carbon::now());
            
            if ($accountAge >= 24) {
                $baseLimit *= 1.5;
            } elseif ($accountAge >= 12) {
                $baseLimit *= 1.2;
            }
        }
        
        // Check payment history
        $recentPayments = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->where('status', 'Paid')
            ->where('created_at', '>=', Carbon::now()->subMonths(6))
            ->count();
        
        if ($recentPayments >= 6) {
            $baseLimit *= 1.25;
        }
        
        // Check for any late payments
        $hasLatePayments = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->where('status', 'Paid')
            ->where('datepaid', '>', 'duedate')
            ->exists();
        
        if ($hasLatePayments) {
            $baseLimit *= 0.75;
        }
        
        $recommendedLimit = round($baseLimit, -2);
        
        return [
            'user_id' => $userId,
            'current_limit' => $profile->credit_limit ?? 0,
            'recommended_limit' => $recommendedLimit,
            'adjustment_percentage' => $profile->credit_limit > 0 
                ? round((($recommendedLimit - $profile->credit_limit) / $profile->credit_limit) * 100, 2) 
                : 100,
            'factors' => $this->getLimitFactors($userId),
        ];
    }
    
    protected function getLimitFactors($userId) {
        return [
            'account_age_months' => Carbon::parse(
                Capsule::table('tblclients')->where('id', $userId)->value('datecreated')
            )->diffInMonths(Carbon::now()),
            'recent_payment_count' => Capsule::table('tblinvoices')
                ->where('userid', $userId)
                ->where('status', 'Paid')
                ->where('created_at', '>=', Carbon::now()->subMonths(6))
                ->count(),
            'current_utilization' => Capsule::table('mod_credit_profiles')
                ->where('user_id', $userId)
                ->value('credit_utilization'),
        ];
    }
    
    public function updateLimit($userId, $newLimit, $reason = null) {
        $profile = Capsule::table('mod_credit_profiles')->where('user_id', $userId)->first();
        
        $oldLimit = $profile ? $profile->credit_limit : 0;
        
        Capsule::table('mod_credit_profiles')
            ->where('user_id', $userId)
            ->update([
                'credit_limit' => $newLimit,
                'available_credit' => $newLimit - ($profile->available_credit - $profile->credit_limit),
            ]);
        
        Capsule::table('mod_credit_history')->insert([
            'user_id' => $userId,
            'event_type' => $newLimit > $oldLimit ? 'limit_increase' : 'limit_decrease',
            'previous_value' => $oldLimit,
            'new_value' => $newLimit,
            'credit_limit_change' => $newLimit - $oldLimit,
            'trigger_reason' => $reason ?? 'Manual adjustment',
        ]);
        
        return [
            'success' => true,
            'user_id' => $userId,
            'old_limit' => $oldLimit,
            'new_limit' => $newLimit,
            'change_amount' => $newLimit - $oldLimit,
        ];
    }
}
```

### lib/CreditDecisionEngine.php

```php
<?php
/**
 * Credit Decision Engine
 */

namespace WHMCS\Module\CreditScoring;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;
use WHMCS\Module\CreditScoring\CreditScorer;
use WHMCS\Module\CreditScoring\CreditLimitManager;

class CreditDecisionEngine {
    
    protected $scorer;
    protected $limitManager;
    protected $minScore;
    protected $autoApproveLimit;
    
    public function __construct() {
        $this->scorer = new CreditScorer();
        $this->limitManager = new CreditLimitManager();
        
        $config = \App::get_config('credit_scoring') ?? [];
        $this->minScore = $config['min_score_threshold'] ?? 500;
        $this->autoApproveLimit = $config['auto_approve_limit'] ?? 500;
    }
    
    public function makeDecision($userId, $type, $amount = null) {
        $profile = $this->scorer->getProfile($userId);
        $score = $profile ? $profile->credit_score : 300;
        
        $decision = $this->evaluateDecision($userId, $type, $amount, $profile);
        
        // Store decision
        $decisionId = Capsule::table('mod_credit_decisions')->insertGetId([
            'user_id' => $userId,
            'decision_type' => $type,
            'requested_amount' => $amount,
            'decision' => $decision['decision'],
            'decision_score' => $score,
            'risk_assessment' => json_encode($decision['risk_factors']),
            'reviewed_at' => Carbon::now(),
            'created_at' => Carbon::now(),
        ]);
        
        return array_merge(['decision_id' => $decisionId], $decision);
    }
    
    protected function evaluateDecision($userId, $type, $amount, $profile) {
        $score = $profile ? $profile->credit_score : 300;
        $riskFactors = [];
        
        // Base decision on score
        if ($score < 300) {
            return [
                'decision' => 'denied',
                'reason' => 'No credit history or critical risk level',
                'risk_factors' => ['critical_risk' => true],
            ];
        }
        
        // Check for overdue invoices
        $overdueCount = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->where('status', 'Overdue')
            ->count();
        
        if ($overdueCount > 3) {
            $riskFactors['high_overdue_count'] = $overdueCount;
            if ($overdueCount > 5) {
                return [
                    'decision' => 'denied',
                    'reason' => "High number of overdue invoices: $overdueCount",
                    'risk_factors' => $riskFactors,
                ];
            }
        }
        
        // Check requested amount vs available credit
        if ($amount && $profile) {
            $newUtilization = ($profile->credit_limit > 0) 
                ? (($profile->available_credit - $amount) / $profile->credit_limit) * 100 
                : 100;
            
            if ($newUtilization < 50) {
                $riskFactors['utilization_ok'] = true;
            } elseif ($newUtilization < 75) {
                $riskFactors['utilization_warning'] = true;
            } else {
                $riskFactors['utilization_high'] = true;
            }
        }
        
        // Auto-approve logic based on score and amount
        if ($score >= $this->minScore) {
            if ($amount && $amount <= $this->autoApproveLimit) {
                return [
                    'decision' => 'approved',
                    'reason' => 'Auto-approved based on credit score and amount',
                    'risk_factors' => $riskFactors,
                ];
            }
            
            if ($score >= 700) {
                return [
                    'decision' => 'approved',
                    'reason' => 'Excellent credit score',
                    'risk_factors' => $riskFactors,
                ];
            }
            
            return [
                'decision' => 'approved',
                'reason' => 'Credit score meets minimum threshold',
                'risk_factors' => $riskFactors,
            ];
        }
        
        if ($score >= 400) {
            return [
                'decision' => 'manual_review',
                'reason' => 'Credit score requires manual review',
                'risk_factors' => $riskFactors,
            ];
        }
        
        return [
            'decision' => 'denied',
            'reason' => 'Credit score below minimum threshold',
            'risk_factors' => array_merge($riskFactors, ['low_score' => $score]),
        ];
    }
    
    public function getDecisionHistory($userId, $limit = 10) {
        return Capsule::table('mod_credit_decisions')
            ->where('user_id', $userId)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get();
    }
}
```

## API Endpoints

```
GET  /api/v1/credit/profile/{userId}       - Get credit profile
GET  /api/v1/credit/score/{userId}         - Calculate/get credit score
GET  /api/v1/credit/history/{userId}       - Get credit history
GET  /api/v1/credit/recommendations/{userId} - Get credit recommendations
POST /api/v1/credit/decision               - Make credit decision
GET  /api/v1/credit/decisions/{userId}      - Get decision history
POST /api/v1/credit/limit/{userId}          - Update credit limit
```

## Hooks Integration

```php
// Auto-check credit before new order
add_hook('ShoppingCartValidateCheckout', 1, function($params) {
    $engine = new \WHMCS\Module\CreditScoring\CreditDecisionEngine();
    $result = $engine->makeDecision($params['user_id'], 'new_order', $params['total']);
    
    if ($result['decision'] === 'denied') {
        return ['error' => 'Credit approval required. Please contact support.'];
    }
});

// Update credit score after payment
add_hook('InvoicePayment', 1, function($params) {
    $scorer = new \WHMCS\Module\CreditScoring\CreditScorer();
    $scorer->calculateScore($params['user_id']);
});

// Notify on credit limit change
add_hook('CreditLimitUpdated', 1, function($params) {
    sendEmailTemplate($params['user_id'], 'credit_limit_change', [
        'old_limit' => $params['old_limit'],
        'new_limit' => $params['new_limit'],
    ]);
});
```
