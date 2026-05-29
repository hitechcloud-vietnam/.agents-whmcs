# WHMCS SLA Management DevKit

## Overview

A comprehensive Service Level Agreement (SLA) management system for WHMCS that tracks, monitors, and enforces service level commitments, calculates uptime, generates SLA reports, and manages credits for missed SLAs.

## Features

- SLA tier definition and management
- Uptime monitoring and tracking
- Downtime incident logging
- Credit calculation and auto-credits
- SLA breach notifications
- Customer SLA portal
- Multi-tier SLA support
- Response time tracking

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_sla_tiers` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `tier_name` VARCHAR(255) NOT NULL,
    `tier_code` VARCHAR(50) NOT NULL,
    `description` TEXT NULL,
    `uptime_percentage` DECIMAL(5,2) NOT NULL DEFAULT 99.9,
    `response_time_minutes` INT UNSIGNED NOT NULL DEFAULT 60,
    `resolution_time_minutes` INT UNSIGNED NOT NULL DEFAULT 480,
    `monthly_credit_percentage` DECIMAL(5,2) NOT NULL DEFAULT 5.00,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `sort_order` INT UNSIGNED NOT NULL DEFAULT 0,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_tier_code` (`tier_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_sla_agreements` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `user_id` INT UNSIGNED NOT NULL,
    `hosting_id` INT UNSIGNED NOT NULL,
    `sla_tier_id` INT UNSIGNED NOT NULL,
    `start_date` DATE NOT NULL,
    `end_date` DATE NULL,
    `renewal_date` DATE NULL,
    `uptime_target` DECIMAL(5,2) NOT NULL DEFAULT 99.9,
    `response_time_target` INT UNSIGNED NOT NULL DEFAULT 60,
    `resolution_time_target` INT UNSIGNED NOT NULL DEFAULT 480,
    `monthly_credit_rate` DECIMAL(5,2) NOT NULL DEFAULT 5.00,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `grace_period_days` INT UNSIGNED NOT NULL DEFAULT 0,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_hosting_sla` (`hosting_id`),
    INDEX `idx_user_id` (`user_id`),
    INDEX `idx_sla_tier_id` (`sla_tier_id`),
    CONSTRAINT `fk_agreement_user` FOREIGN KEY (`user_id`) REFERENCES `tblusers`(`id`) ON DELETE CASCADE,
    CONSTRAINT `fk_agreement_tier` FOREIGN KEY (`sla_tier_id`) REFERENCES `mod_sla_tiers`(`id`) ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_sla_incidents` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `sla_agreement_id` INT UNSIGNED NOT NULL,
    `incident_type` ENUM('downtime', 'slow_response', 'missed_resolution', 'other') NOT NULL DEFAULT 'downtime',
    `severity` ENUM('critical', 'major', 'minor') NOT NULL DEFAULT 'major',
    `title` VARCHAR(255) NOT NULL,
    `description` TEXT NULL,
    `started_at` DATETIME NOT NULL,
    `resolved_at` DATETIME NULL,
    `duration_minutes` INT UNSIGNED NOT NULL DEFAULT 0,
    `acknowledged_at` DATETIME NULL,
    `acknowledged_by` INT UNSIGNED NULL,
    `handled_by` INT UNSIGNED NULL,
    `credit_issued` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    `credit_description` TEXT NULL,
    `is_billable` TINYINT(1) NOT NULL DEFAULT 0,
    `status` ENUM('open', 'acknowledged', 'investigating', 'resolved', 'closed') NOT NULL DEFAULT 'open',
    `internal_notes` TEXT NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_agreement_id` (`sla_agreement_id`),
    INDEX `idx_started_at` (`started_at`),
    INDEX `idx_status` (`status`),
    CONSTRAINT `fk_incident_agreement` FOREIGN KEY (`sla_agreement_id`) REFERENCES `mod_sla_agreements`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_sla_metrics` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `sla_agreement_id` INT UNSIGNED NOT NULL,
    `metric_date` DATE NOT NULL,
    `uptime_percentage` DECIMAL(5,2) NOT NULL DEFAULT 100.00,
    `downtime_minutes` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    `avg_response_time` DECIMAL(10,2) NULL,
    `avg_resolution_time` DECIMAL(10,2) NULL,
    `tickets_created` INT UNSIGNED NOT NULL DEFAULT 0,
    `tickets_responded` INT UNSIGNED NOT NULL DEFAULT 0,
    `tickets_resolved` INT UNSIGNED NOT NULL DEFAULT 0,
    `sla_met_count` INT UNSIGNED NOT NULL DEFAULT 0,
    `sla_missed_count` INT UNSIGNED NOT NULL DEFAULT 0,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_agreement_date` (`sla_agreement_id`, `metric_date`),
    CONSTRAINT `fk_metric_agreement` FOREIGN KEY (`sla_agreement_id`) REFERENCES `mod_sla_agreements`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Files

### sla_management.php (Main Module)

```php
<?php
/**
 * WHMCS SLA Management Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/SLAManager.php';
require_once __DIR__ . '/lib/UptimeTracker.php';
require_once __DIR__ . '/lib/CreditCalculator.php';

use WHMCS\Module\SlaManagement\SLAManager;
use WHMCS\Module\SlaManagement\UptimeTracker;
use WHMCS\Module\SlaManagement\CreditCalculator;

/**
 * Activate Module
 */
function whmcs_sla_management_activate() {
    $manager = new SLAManager();
    return $manager->activate();
}

/**
 * Deactivate Module
 */
function whmcs_sla_management_deactivate() {
    return ['success' => true, 'msg' => 'SLA Management module deactivated'];
}

/**
 * Upgrade Module
 */
function whmcs_sla_management_upgrade($version) {
    $manager = new SLAManager();
    return $manager->upgrade($version);
}

/**
 * Module Configuration
 */
function whmcs_sla_management_config() {
    return [
        'name' => [
            'FriendlyName' => 'SLA Management',
            'Type' => 'System',
            'Value' => 'SLA Management Module v1.0.0',
        ],
        'default_tier' => [
            'FriendlyName' => 'Default SLA Tier',
            'Type' => 'dropdown',
            'Options' => [
                'basic' => 'Basic (99.5%)',
                'standard' => 'Standard (99.9%)',
                'premium' => 'Premium (99.99%)',
            ],
            'Default' => 'standard',
        ],
        'auto_credits' => [
            'FriendlyName' => 'Auto-issue Credits',
            'Type' => 'yesno',
            'Description' => 'Automatically issue SLA credits when targets are missed',
        ],
        'credit_calculation_day' => [
            'FriendlyName' => 'Credit Calculation Day',
            'Type' => 'dropdown',
            'Options' => [
                '1' => '1st of month',
                '5' => '5th of month',
                '15' => '15th of month',
            ],
            'Default' => '1',
        ],
        'email_notifications' => [
            'FriendlyName' => 'Enable Email Notifications',
            'Type' => 'yesno',
        ],
    ];
}

// Hooks
add_hook('DailyCronJob', 1, function() {
    $tracker = new UptimeTracker();
    $calculator = new CreditCalculator();
    
    $tracker->calculateDailyUptime();
    $calculator->processMonthlyCredits();
});

add_hook('TicketOpen', 1, function($params) {
    $manager = new SLAManager();
    $manager->trackResponseTime($params['ticket_id']);
});

add_hook('TicketReply', 1, function($params) {
    $manager = new SLAManager();
    $manager->trackResponseTime($params['ticket_id'], 'first_response');
});

add_hook('ServiceTerminated', 1, function($params) {
    $manager = new SLAManager();
    $manager->deactivateAgreement($params['service_id']);
});
```

### lib/SLAManager.php

```php
<?php
/**
 * SLA Manager
 */

namespace WHMCS\Module\SlaManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class SLAManager {
    
    protected $version = '1.0.0';
    
    public function activate() {
        try {
            $this->createTables();
            $this->createDefaultTiers();
            return ['success' => true, 'msg' => 'SLA Management module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_sla_tiers` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `tier_name` VARCHAR(255) NOT NULL,
                `tier_code` VARCHAR(50) NOT NULL,
                `description` TEXT NULL,
                `uptime_percentage` DECIMAL(5,2) NOT NULL DEFAULT 99.9,
                `response_time_minutes` INT UNSIGNED NOT NULL DEFAULT 60,
                `resolution_time_minutes` INT UNSIGNED NOT NULL DEFAULT 480,
                `monthly_credit_percentage` DECIMAL(5,2) NOT NULL DEFAULT 5.00,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                `sort_order` INT UNSIGNED NOT NULL DEFAULT 0,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_tier_code` (`tier_code`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_sla_agreements` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL,
                `hosting_id` INT UNSIGNED NOT NULL,
                `sla_tier_id` INT UNSIGNED NOT NULL,
                `start_date` DATE NOT NULL,
                `end_date` DATE NULL,
                `renewal_date` DATE NULL,
                `uptime_target` DECIMAL(5,2) NOT NULL DEFAULT 99.9,
                `response_time_target` INT UNSIGNED NOT NULL DEFAULT 60,
                `resolution_time_target` INT UNSIGNED NOT NULL DEFAULT 480,
                `monthly_credit_rate` DECIMAL(5,2) NOT NULL DEFAULT 5.00,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                `grace_period_days` INT UNSIGNED NOT NULL DEFAULT 0,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_hosting_sla` (`hosting_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_sla_incidents` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `sla_agreement_id` INT UNSIGNED NOT NULL,
                `incident_type` ENUM('downtime', 'slow_response', 'missed_resolution', 'other') NOT NULL DEFAULT 'downtime',
                `severity` ENUM('critical', 'major', 'minor') NOT NULL DEFAULT 'major',
                `title` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `started_at` DATETIME NOT NULL,
                `resolved_at` DATETIME NULL,
                `duration_minutes` INT UNSIGNED NOT NULL DEFAULT 0,
                `acknowledged_at` DATETIME NULL,
                `handled_by` INT UNSIGNED NULL,
                `credit_issued` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
                `is_billable` TINYINT(1) NOT NULL DEFAULT 0,
                `status` ENUM('open', 'acknowledged', 'investigating', 'resolved', 'closed') NOT NULL DEFAULT 'open',
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_sla_metrics` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `sla_agreement_id` INT UNSIGNED NOT NULL,
                `metric_date` DATE NOT NULL,
                `uptime_percentage` DECIMAL(5,2) NOT NULL DEFAULT 100.00,
                `downtime_minutes` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
                `avg_response_time` DECIMAL(10,2) NULL,
                `avg_resolution_time` DECIMAL(10,2) NULL,
                `sla_met_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `sla_missed_count` INT UNSIGNED NOT NULL DEFAULT 0,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_agreement_date` (`sla_agreement_id`, `metric_date`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function createDefaultTiers() {
        $tiers = [
            [
                'tier_name' => 'Basic',
                'tier_code' => 'basic',
                'description' => 'Basic SLA tier with 99.5% uptime',
                'uptime_percentage' => 99.5,
                'response_time_minutes' => 120,
                'resolution_time_minutes' => 1440,
                'monthly_credit_percentage' => 5.00,
                'sort_order' => 1,
            ],
            [
                'tier_name' => 'Standard',
                'tier_code' => 'standard',
                'description' => 'Standard SLA tier with 99.9% uptime',
                'uptime_percentage' => 99.9,
                'response_time_minutes' => 60,
                'resolution_time_minutes' => 480,
                'monthly_credit_percentage' => 10.00,
                'sort_order' => 2,
            ],
            [
                'tier_name' => 'Premium',
                'tier_code' => 'premium',
                'description' => 'Premium SLA tier with 99.99% uptime',
                'uptime_percentage' => 99.99,
                'response_time_minutes' => 15,
                'resolution_time_minutes' => 120,
                'monthly_credit_percentage' => 25.00,
                'sort_order' => 3,
            ],
        ];
        
        foreach ($tiers as $tier) {
            if (!Capsule::table('mod_sla_tiers')->where('tier_code', $tier['tier_code'])->exists()) {
                Capsule::table('mod_sla_tiers')->insert($tier);
            }
        }
    }
    
    /**
     * Create SLA agreement for a service
     */
    public function createAgreement($userId, $hostingId, $tierCode, $startDate = null, $endDate = null) {
        $tier = Capsule::table('mod_sla_tiers')->where('tier_code', $tierCode)->first();
        
        if (!$tier) {
            return ['success' => false, 'msg' => 'SLA tier not found'];
        }
        
        $existing = Capsule::table('mod_sla_agreements')->where('hosting_id', $hostingId)->first();
        if ($existing) {
            return ['success' => false, 'msg' => 'SLA agreement already exists for this service'];
        }
        
        $agreementId = Capsule::table('mod_sla_agreements')->insertGetId([
            'user_id' => $userId,
            'hosting_id' => $hostingId,
            'sla_tier_id' => $tier->id,
            'start_date' => $startDate ?? Carbon::today(),
            'end_date' => $endDate,
            'uptime_target' => $tier->uptime_percentage,
            'response_time_target' => $tier->response_time_minutes,
            'resolution_time_target' => $tier->resolution_time_minutes,
            'monthly_credit_rate' => $tier->monthly_credit_percentage,
            'is_active' => 1,
        ]);
        
        return ['success' => true, 'msg' => 'SLA agreement created', 'agreement_id' => $agreementId];
    }
    
    /**
     * Log an incident
     */
    public function logIncident($agreementId, $type, $title, $startedAt = null) {
        $incidentId = Capsule::table('mod_sla_incidents')->insertGetId([
            'sla_agreement_id' => $agreementId,
            'incident_type' => $type,
            'title' => $title,
            'started_at' => $startedAt ?? Carbon::now(),
            'status' => 'open',
        ]);
        
        return ['success' => true, 'incident_id' => $incidentId];
    }
    
    /**
     * Resolve an incident
     */
    public function resolveIncident($incidentId) {
        $incident = Capsule::table('mod_sla_incidents')->where('id', $incidentId)->first();
        
        if (!$incident) {
            return ['success' => false, 'msg' => 'Incident not found'];
        }
        
        $durationMinutes = Carbon::parse($incident->started_at)->diffInMinutes(Carbon::now());
        
        Capsule::table('mod_sla_incidents')
            ->where('id', $incidentId)
            ->update([
                'resolved_at' => Carbon::now(),
                'duration_minutes' => $durationMinutes,
                'status' => 'resolved',
            ]);
        
        return ['success' => true, 'duration_minutes' => $durationMinutes];
    }
    
    /**
     * Get SLA status for a service
     */
    public function getSlaStatus($hostingId) {
        $agreement = Capsule::table('mod_sla_agreements')
            ->join('mod_sla_tiers', 'mod_sla_agreements.sla_tier_id', '=', 'mod_sla_tiers.id')
            ->where('mod_sla_agreements.hosting_id', $hostingId)
            ->where('mod_sla_agreements.is_active', 1)
            ->select('mod_sla_agreements.*', 'mod_sla_tiers.tier_name', 'mod_sla_tiers.tier_code')
            ->first();
        
        if (!$agreement) {
            return null;
        }
        
        $today = Carbon::today();
        $monthStart = $today->copy()->startOfMonth();
        $monthEnd = $today->copy()->endOfMonth();
        
        $monthMetrics = Capsule::table('mod_sla_metrics')
            ->where('sla_agreement_id', $agreement->id)
            ->whereBetween('metric_date', [$monthStart->toDateString(), $monthEnd->toDateString()])
            ->selectRaw('AVG(uptime_percentage) as avg_uptime, SUM(downtime_minutes) as total_downtime')
            ->first();
        
        return [
            'agreement_id' => $agreement->id,
            'tier_name' => $agreement->tier_name,
            'tier_code' => $agreement->tier_code,
            'start_date' => $agreement->start_date,
            'end_date' => $agreement->end_date,
            'uptime_target' => $agreement->uptime_target,
            'current_uptime' => $monthMetrics->avg_uptime ?? 100.00,
            'total_downtime_minutes' => $monthMetrics->total_downtime ?? 0,
            'uptime_met' => ($monthMetrics->avg_uptime ?? 100) >= $agreement->uptime_target,
        ];
    }
    
    /**
     * Track support ticket response time
     */
    public function trackResponseTime($ticketId, $responseType = 'reply') {
        $ticket = Capsule::table('tbltickets')->where('id', $ticketId)->first();
        if (!$ticket) return;
        
        $service = Capsule::table('tblhosting')->where('id', $ticket->userid)->first();
        if (!$service) return;
        
        $agreement = Capsule::table('mod_sla_agreements')
            ->where('hosting_id', $service->id)
            ->where('is_active', 1)
            ->first();
        
        if (!$agreement) return;
        
        $firstResponse = $responseType === 'first_response' 
            ? Carbon::parse($ticket->created_at)->diffInMinutes(Carbon::now())
            : 0;
        
        $logTable = Capsule::table('mod_sla_ticket_log') ?? null;
    }
    
    public function deactivateAgreement($hostingId) {
        return Capsule::table('mod_sla_agreements')
            ->where('hosting_id', $hostingId)
            ->update([
                'is_active' => 0,
                'end_date' => Carbon::today(),
            ]);
    }
}
```

### lib/UptimeTracker.php

```php
<?php
/**
 * Uptime Tracker
 */

namespace WHMCS\Module\SlaManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class UptimeTracker {
    
    /**
     * Calculate daily uptime for all agreements
     */
    public function calculateDailyUptime() {
        $agreements = Capsule::table('mod_sla_agreements')
            ->where('is_active', 1)
            ->get();
        
        $today = Carbon::today();
        
        foreach ($agreements as $agreement) {
            $this->calculateAgreementUptime($agreement->id, $today);
        }
    }
    
    /**
     * Calculate uptime for a specific agreement
     */
    public function calculateAgreementUptime($agreementId, $date) {
        $agreement = Capsule::table('mod_sla_agreements')->where('id', $agreementId)->first();
        
        $startOfDay = Carbon::parse($date)->startOfDay();
        $endOfDay = Carbon::parse($date)->endOfDay();
        
        $incidents = Capsule::table('mod_sla_incidents')
            ->where('sla_agreement_id', $agreementId)
            ->where(function($query) use ($startOfDay, $endOfDay) {
                $query->whereBetween('started_at', [$startOfDay, $endOfDay]);
            })
            ->get();
        
        $totalDowntimeMinutes = 0;
        
        foreach ($incidents as $incident) {
            $incidentStart = Carbon::parse($incident->started_at);
            $incidentEnd = $incident->resolved_at 
                ? Carbon::parse($incident->resolved_at) 
                : Carbon::now();
            
            // Clamp to day boundaries
            $from = max($incidentStart, $startOfDay);
            $to = min($incidentEnd, $endOfDay);
            
            if ($to > $from) {
                $totalDowntimeMinutes += $from->diffInMinutes($to);
            }
        }
        
        $totalMinutesInDay = 24 * 60;
        $uptimePercentage = $totalMinutesInDay > 0 
            ? (($totalMinutesInDay - $totalDowntimeMinutes) / $totalMinutesInDay) * 100 
            : 100;
        
        Capsule::table('mod_sla_metrics')->updateOrInsert(
            ['sla_agreement_id' => $agreementId, 'metric_date' => $date->toDateString()],
            [
                'uptime_percentage' => min(100, max(0, $uptimePercentage)),
                'downtime_minutes' => $totalDowntimeMinutes,
            ]
        );
    }
    
    /**
     * Get uptime for a period
     */
    public function getUptimeForPeriod($agreementId, Carbon $startDate, Carbon $endDate) {
        $metrics = Capsule::table('mod_sla_metrics')
            ->where('sla_agreement_id', $agreementId)
            ->whereBetween('metric_date', [$startDate->toDateString(), $endDate->toDateString()])
            ->selectRaw('AVG(uptime_percentage) as avg_uptime, SUM(downtime_minutes) as total_downtime')
            ->first();
        
        return [
            'average_uptime' => round($metrics->avg_uptime ?? 100, 2),
            'total_downtime_minutes' => $metrics->total_downtime ?? 0,
        ];
    }
}
```

### lib/CreditCalculator.php

```php
<?php
/**
 * SLA Credit Calculator
 */

namespace WHMCS\Module\SlaManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class CreditCalculator {
    
    /**
     * Process monthly credits for all agreements
     */
    public function processMonthlyCredits() {
        $calculationDay = (int)(\App::get_config('sla_management')['credit_calculation_day'] ?? 1);
        $today = Carbon::today();
        
        if ($today->day !== $calculationDay) {
            return;
        }
        
        $lastMonth = $today->copy()->subMonth();
        
        $agreements = Capsule::table('mod_sla_agreements')
            ->where('is_active', 1)
            ->where('start_date', '<=', $lastMonth->endOfMonth()->toDateString())
            ->get();
        
        foreach ($agreements as $agreement) {
            $this->calculateAndIssueCredit($agreement->id, $lastMonth);
        }
    }
    
    /**
     * Calculate and issue credit for an agreement
     */
    public function calculateAndIssueCredit($agreementId, Carbon $month) {
        $agreement = Capsule::table('mod_sla_agreements')->where('id', $agreementId)->first();
        
        $monthStart = $month->copy()->startOfMonth();
        $monthEnd = $month->copy()->endOfMonth();
        
        $metrics = Capsule::table('mod_sla_metrics')
            ->where('sla_agreement_id', $agreementId)
            ->whereBetween('metric_date', [$monthStart->toDateString(), $monthEnd->toDateString()])
            ->selectRaw('AVG(uptime_percentage) as avg_uptime')
            ->first();
        
        $actualUptime = $metrics->avg_uptime ?? 100;
        $targetUptime = $agreement->uptime_target;
        
        if ($actualUptime >= $targetUptime) {
            return ['success' => true, 'credit' => 0, 'reason' => 'SLA met'];
        }
        
        $shortfall = $targetUptime - $actualUptime;
        $creditPercentage = min($agreement->monthly_credit_rate, ($shortfall / $targetUptime) * 100);
        
        $service = Capsule::table('tblhosting')->where('id', $agreement->hosting_id)->first();
        $monthlyAmount = $service->amount ?? 0;
        $creditAmount = ($monthlyAmount * $creditPercentage) / 100;
        
        if ($creditAmount > 0) {
            $this->issueCredit($agreement->id, $creditAmount, $month->format('F Y'), $actualUptime, $targetUptime);
        }
        
        return [
            'success' => true,
            'credit' => $creditAmount,
            'uptime_shortfall' => $shortfall,
        ];
    }
    
    /**
     * Issue credit to customer
     */
    protected function issueCredit($agreementId, $amount, $period, $actualUptime, $targetUptime) {
        Capsule::table('mod_sla_incidents')->insert([
            'sla_agreement_id' => $agreementId,
            'incident_type' => 'downtime',
            'title' => 'Monthly SLA Credit - Uptime Below Target',
            'description' => "Credit issued for $period. Actual uptime: $actualUptime%, Target: $targetUptime%",
            'started_at' => Carbon::parse($period . ' 00:00:00'),
            'resolved_at' => Carbon::now(),
            'credit_issued' => $amount,
            'status' => 'closed',
        ]);
        
        Capsule::table('tblcredit')->insert([
            'userid' => Capsule::table('mod_sla_agreements')->where('id', $agreementId)->first()->user_id,
            'amount' => $amount,
            'description' => "SLA Credit for $period (Uptime: {$actualUptime}%)",
            'date' => Carbon::now(),
        ]);
    }
}
```

## API Endpoints

```
GET  /api/v1/sla/tiers               - List all SLA tiers
GET  /api/v1/sla/agreements/{hostingId} - Get SLA agreement
POST /api/v1/sla/agreements          - Create SLA agreement
PUT  /api/v1/sla/agreements/{id}     - Update SLA agreement
POST /api/v1/sla/incidents           - Log new incident
PUT  /api/v1/sla/incidents/{id}      - Update incident
POST /api/v1/sla/incidents/{id}/resolve - Resolve incident
GET  /api/v1/sla/metrics/{agreementId} - Get SLA metrics
GET  /api/v1/sla/uptime/{agreementId}   - Get uptime data
POST /api/v1/sla/credits/calculate   - Calculate credits
```

## Hooks Integration

```php
// Track SLA when service is created
add_hook('ServiceAdd', 1, function($params) {
    $slaManager = new \WHMCS\Module\SlaManagement\SLAManager();
    $defaultTier = \App::get_config('sla_management')['default_tier'] ?? 'standard';
    $slaManager->createAgreement($params['user_id'], $params['service_id'], $defaultTier);
});

// Log downtime when server goes down
add_hook('ServerDown', 1, function($params) {
    $slaManager = new \WHMCS\Module\SlaManagement\SLAManager();
    $services = Capsule::table('tblhosting')->where('serverid', $params['server_id'])->get();
    
    foreach ($services as $service) {
        $agreement = Capsule::table('mod_sla_agreements')->where('hosting_id', $service->id)->first();
        if ($agreement) {
            $slaManager->logIncident($agreement->id, 'downtime', "Server {$params['server_name']} is down");
        }
    }
});

// Issue credit when SLA is missed
add_hook('InvoiceCreationprecreation', 1, function($params) {
    $calculator = new \WHMCS\Module\SlaManagement\CreditCalculator();
    $credit = $calculator->calculateAndIssueCredit($params['agreement_id'], Carbon::now()->subMonth());
    
    if ($credit['credit'] > 0) {
        // Add credit to invoice
    }
});
```
