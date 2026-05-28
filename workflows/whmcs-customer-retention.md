# WHMCS Customer Retention Workflow

## Purpose

Implement comprehensive customer retention programs for WHMCS to build long-term customer relationships, increase lifetime value, and reduce churn. This workflow covers loyalty programs, engagement strategies, and retention measurement.

## Prerequisites

- WHMCS v8.0+
- Admin access to WHMCS
- Communication system (email/API)
- Customer lifecycle data

## Workflow Steps

### Step 1: Retention Strategy Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                 Customer Retention Framework                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Retention Pillars:                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Value     │  │   Experience │  │   Relationship│         │
│  │  Delivery   │  │   Excellence │  │    Building  │          │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
│  │ • ROI       │  │ • Onboarding │  │ • Personal   │          │
│  │ • Support   │  │ • Self-serve │  │ • Anniversary│          │
│  │ • Updates   │  │ • Performance│  │ • Loyalty    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  Customer Lifecycle:                                             │
│                                                                  │
│  Acquire ──▶ Onboard ──▶ Engage ──▶ Retain ──▶ Advocate       │
│     │           │           │           │           │           │
│     ▼           ▼           ▼           ▼           ▼           │
│  Welcome    Tutorial    Regular    Loyalty    Referral        │
│  Kit        Setup        Check-in   Program    Program        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: Retention Program Implementation

```php
<?php
// /var/www/html/whmcs/includes/helpers/RetentionManager.php

namespace WHMCS\Helpers;

use Illuminate\Database\Capsule\Manager as Capsule;

class RetentionManager
{
    /**
     * Get client retention score
     */
    public static function getRetentionScore(int $clientId): array
    {
        $client = \WHMCS\User\Client::find($clientId);
        
        return [
            'tenure_score' => self::calculateTenureScore($client),
            'engagement_score' => self::calculateEngagementScore($clientId),
            'satisfaction_score' => self::calculateSatisfactionScore($clientId),
            'loyalty_score' => self::calculateLoyaltyScore($clientId),
            'overall_score' => 0 // Calculated below
        ];
    }
    
    /**
     * Calculate tenure score (longer = better)
     */
    private static function calculateTenureScore(object $client): float
    {
        $yearsActive = (time() - strtotime($client->createdat)) / (365 * 24 * 3600);
        
        // Score: 0-100 based on years (cap at 10 years = 100)
        return round(min(100, $yearsActive * 10), 2);
    }
    
    /**
     * Calculate engagement score
     */
    private static function calculateEngagementScore(int $clientId): float
    {
        $activities = Capsule::table('tblactivitylog')
            ->where('userid', $clientId)
            ->where('date', '>=', date('Y-m-d', strtotime('-90 days')))
            ->count();
        
        $tickets = Capsule::table('tbltickets')
            ->where('userid', $clientId)
            ->where('created', '>=', date('Y-m-d', strtotime('-90 days')))
            ->count();
        
        // Normalize to 0-100
        $activityScore = min(100, $activities * 2);
        $supportScore = min(50, $tickets * 5); // Tickets indicate engagement but also issues
        
        return round($activityScore + $supportScore, 2);
    }
    
    /**
     * Calculate satisfaction score (based on tickets and feedback)
     */
    private static function calculateSatisfactionScore(int $clientId): float
    {
        // Get recent ticket satisfaction ratings
        $ratings = Capsule::table('tblticketfeedback')
            ->where('userid', $clientId)
            ->where('created', '>=', date('Y-m-d', strtotime('-180 days')))
            ->avg('rating');
        
        if ($ratings === null) {
            // Default to neutral if no feedback
            return 75;
        }
        
        // Convert 1-5 rating to 0-100 score
        return round(($ratings / 5) * 100, 2);
    }
    
    /**
     * Calculate loyalty score
     */
    private static function calculateLoyaltyScore(int $clientId): float
    {
        // Services retention
        $totalServices = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->count();
        
        $activeServices = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->where('domainstatus', 'Active')
            ->count();
        
        $retentionRate = $totalServices > 0 ? ($activeServices / $totalServices) * 100 : 0;
        
        // Renewal rate
        $renewals = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->where('domainstatus', 'Active')
            ->where('billingcycle', '>', 0) // Non-free
            ->count();
        
        // Calculate loyalty
        $loyalty = ($retentionRate * 0.7) + (min(100, $renewals * 10) * 0.3);
        
        return round($loyalty, 2);
    }
    
    /**
     * Get retention status
     */
    public static function getRetentionStatus(int $clientId): string
    {
        $scores = self::getRetentionScore($clientId);
        
        $overall = ($scores['tenure_score'] * 0.2) + 
                   ($scores['engagement_score'] * 0.3) + 
                   ($scores['satisfaction_score'] * 0.25) + 
                   ($scores['loyalty_score'] * 0.25);
        
        if ($overall >= 80) {
            return 'champion';
        } elseif ($overall >= 60) {
            return 'loyal';
        } elseif ($overall >= 40) {
            return 'neutral';
        } elseif ($overall >= 20) {
            return 'at_risk';
        } else {
            return 'churned';
        }
    }
}
```

### Step 3: Loyalty Program Implementation

```php
<?php
// /var/www/html/whmcs/includes/helpers/LoyaltyProgram.php

namespace WHMCS\Helpers;

class LoyaltyProgram
{
    private $tiers = [
        'bronze' => [
            'min_points' => 0,
            'discount' => 0,
            'benefits' => ['basic_support']
        ],
        'silver' => [
            'min_points' => 500,
            'discount' => 5,
            'benefits' => ['basic_support', 'priority_tickets']
        ],
        'gold' => [
            'min_points' => 2000,
            'discount' => 10,
            'benefits' => ['priority_support', 'early_access', 'monthly_credits']
        ],
        'platinum' => [
            'min_points' => 5000,
            'discount' => 15,
            'benefits' => ['dedicated_support', 'beta_access', 'quarterly_credits', 'annual_review']
        ]
    ];
    
    /**
     * Award points to client
     */
    public static function awardPoints(int $clientId, string $action, int $basePoints = 10): bool
    {
        $multiplier = self::getActionMultiplier($action);
        $points = $basePoints * $multiplier;
        
        // Add points
        Capsule::table('tblloyalty_points')->insert([
            'client_id' => $clientId,
            'action' => $action,
            'points' => $points,
            'created_at' => date('Y-m-d H:i:s')
        ]);
        
        // Update total
        self::updateTotalPoints($clientId);
        
        // Check for tier upgrade
        self::checkTierUpgrade($clientId);
        
        return true;
    }
    
    /**
     * Get action multiplier for points
     */
    private static function getActionMultiplier(string $action): int
    {
        $multipliers = [
            'purchase' => 10,
            'renewal' => 5,
            'referral' => 50,
            'review' => 20,
            'survey' => 10,
            'support_ticket' => 2,
            'login' => 1
        ];
        
        return $multipliers[$action] ?? 1;
    }
    
    /**
     * Update client total points
     */
    private static function updateTotalPoints(int $clientId): void
    {
        $total = Capsule::table('tblloyalty_points')
            ->where('client_id', $clientId)
            ->sum('points');
        
        Capsule::table('tblloyalty_members')
            ->updateOrInsert(
                ['client_id' => $clientId],
                [
                    'total_points' => $total,
                    'updated_at' => date('Y-m-d H:i:s')
                ]
            );
    }
    
    /**
     * Check and apply tier upgrade
     */
    private static function checkTierUpgrade(int $clientId): void
    {
        $member = Capsule::table('tblloyalty_members')
            ->where('client_id', $clientId)
            ->first();
        
        if (!$member) {
            return;
        }
        
        $currentTier = $member->tier;
        $newTier = self::calculateTier($member->total_points);
        
        if ($newTier !== $currentTier) {
            // Tier upgrade
            Capsule::table('tblloyalty_members')
                ->where('client_id', $clientId)
                ->update(['tier' => $newTier]);
            
            // Send notification
            self::sendTierUpgradeNotification($clientId, $newTier);
        }
    }
    
    /**
     * Calculate tier from points
     */
    private static function calculateTier(int $points): string
    {
        if ($points >= $this->tiers['platinum']['min_points']) {
            return 'platinum';
        } elseif ($points >= $this->tiers['gold']['min_points']) {
            return 'gold';
        } elseif ($points >= $this->tiers['silver']['min_points']) {
            return 'silver';
        } else {
            return 'bronze';
        }
    }
    
    /**
     * Get tier benefits
     */
    public static function getTierBenefits(int $clientId): array
    {
        $member = Capsule::table('tblloyalty_members')
            ->where('client_id', $clientId)
            ->first();
        
        if (!$member) {
            return $this->tiers['bronze']['benefits'];
        }
        
        return $this->tiers[$member->tier]['benefits'] ?? $this->tiers['bronze']['benefits'];
    }
    
    /**
     * Send tier upgrade notification
     */
    private static function sendTierUpgradeNotification(int $clientId, string $newTier): void
    {
        $client = \WHMCS\User\Client::find($clientId);
        
        // Send email with benefits
        sendEmail('LoyaltyTierUpgrade', $clientId, [
            'new_tier' => ucfirst($newTier),
            'benefits' => implode(', ', $this->tiers[$newTier]['benefits']),
            'discount' => $this->tiers[$newTier]['discount'] . '%'
        ]);
    }
}
```

### Step 4: Retention Engagement Programs

```php
<?php
// /var/www/html/whmcs/includes/hooks/retention_programs.php

/**
 * Anniversary celebration program
 */
add_hook('DailyCronJob', 1, function() {
    $today = date('m-d');
    
    // Find clients with anniversary today
    $anniversaryClients = Capsule::select("
        SELECT id, firstname, lastname, createdat
        FROM tblclients
        WHERE DATE_FORMAT(createdat, '%m-%d') = ?
        AND status = 'Active'
    ", [$today]);
    
    foreach ($anniversaryClients as $client) {
        $years = floor((time() - strtotime($client->createdat)) / (365 * 24 * 3600));
        
        if ($years >= 1) {
            // Award anniversary bonus points
            \WHMCS\Helpers\LoyaltyProgram::awardPoints($client->id, 'anniversary', $years * 50);
            
            // Send anniversary email
            sendEmail('AnniversaryCelebration', $client->id, [
                'years' => $years,
                'client_name' => $client->firstname
            ]);
            
            logActivity("Anniversary celebration sent to client {$client->id} ({$years} years)");
        }
    }
});

/**
 * Milestone check-in program
 */
add_hook('DailyCronJob', 1, function() {
    // Check for clients at 30, 60, 90 day milestones
    $milestones = [30, 60, 90];
    
    foreach ($milestones as $days) {
        $clients = Capsule::select("
            SELECT id, firstname
            FROM tblclients
            WHERE DATEDIFF(NOW(), createdat) = ?
            AND status = 'Active'
        ", [$days]);
        
        foreach ($clients as $client) {
            sendEmail('MilestoneCheckIn', $client->id, [
                'days' => $days,
                'client_name' => $client->firstname
            ]);
            
            \WHMCS\Helpers\LoyaltyProgram::awardPoints($client->id, 'milestone_checkin', 25);
        }
    }
});

/**
 * Win-back program for inactive clients
 */
add_hook('DailyCronJob', 1, function() {
    // Find clients inactive for 60+ days
    $inactiveClients = Capsule::select("
        SELECT c.id, c.firstname, c.email, MAX(a.date) as last_activity
        FROM tblclients c
        LEFT JOIN tblactivitylog a ON c.id = a.userid
        WHERE c.status = 'Active'
        GROUP BY c.id
        HAVING last_activity < DATE_SUB(NOW(), INTERVAL 60 DAY)
        OR last_activity IS NULL
        LIMIT 50
    ");
    
    foreach ($inactiveClients as $client) {
        // Add to win-back campaign
        Capsule::table('tblretention_campaigns')->insert([
            'client_id' => $client->id,
            'campaign_type' => 'win_back',
            'scheduled_date' => date('Y-m-d', strtotime('+7 days')),
            'status' => 'pending'
        ]);
    }
});
```

### Step 5: Loyalty Program Storage Schema

```sql
-- Create loyalty points table
CREATE TABLE IF NOT EXISTS `tblloyalty_points` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT UNSIGNED NOT NULL,
    `action` VARCHAR(50) NOT NULL,
    `points` INT NOT NULL,
    `description` VARCHAR(255),
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_client_id` (`client_id`),
    INDEX `idx_action` (`action`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Create loyalty members table
CREATE TABLE IF NOT EXISTS `tblloyalty_members` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT UNSIGNED NOT NULL UNIQUE,
    `tier` ENUM('bronze', 'silver', 'gold', 'platinum') DEFAULT 'bronze',
    `total_points` INT DEFAULT 0,
    `lifetime_points` INT DEFAULT 0,
    `tier_achieved_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    `updated_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_tier` (`tier`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Create retention campaigns table
CREATE TABLE IF NOT EXISTS `tblretention_campaigns` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT UNSIGNED NOT NULL,
    `campaign_type` VARCHAR(50) NOT NULL,
    `campaign_name` VARCHAR(100),
    `scheduled_date` DATE,
    `sent_date` TIMESTAMP NULL,
    `status` ENUM('pending', 'sent', 'failed', 'skipped') DEFAULT 'pending',
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_client_campaign` (`client_id`, `campaign_type`),
    INDEX `idx_scheduled` (`scheduled_date`, `status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Create referral tracking table
CREATE TABLE IF NOT EXISTS `tblreferral_tracking` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `referrer_id` INT UNSIGNED NOT NULL,
    `referred_id` INT UNSIGNED,
    `referral_code` VARCHAR(50),
    `status` ENUM('pending', 'completed', 'rewarded') DEFAULT 'pending',
    `reward_points` INT DEFAULT 0,
    `rewarded_at` TIMESTAMP NULL,
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_referrer` (`referrer_id`),
    INDEX `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Step 6: Retention Dashboard Widget

```php
<?php
// /var/www/html/whmcs/modules/widgets/RetentionDashboard.php

namespace WHMCS\Module\Widget;

class RetentionDashboard extends \WHMCS\Module\AbstractWidget
{
    protected $title = 'Customer Retention';
    protected $author = 'System';
    protected $type = 'dashboard';
    
    public function getData(): array
    {
        // Loyalty program stats
        $loyaltyStats = Capsule::select("
            SELECT 
                tier,
                COUNT(*) as members,
                SUM(total_points) as total_points
            FROM tblloyalty_members
            GROUP BY tier
        ");
        
        // Retention rate by tier
        $tierRetention = $this->calculateTierRetention();
        
        // Active campaigns
        $activeCampaigns = Capsule::table('tblretention_campaigns')
            ->where('status', 'pending')
            ->where('scheduled_date', '<=', date('Y-m-d', strtotime('+7 days')))
            ->count();
        
        // Monthly retention trend
        $retentionTrend = $this->getRetentionTrend();
        
        return [
            'loyalty_stats' => $loyaltyStats,
            'tier_retention' => $tierRetention,
            'active_campaigns' => $activeCampaigns,
            'retention_trend' => $retentionTrend,
            'total_members' => array_sum(array_column($loyaltyStats, 'members'))
        ];
    }
    
    private function calculateTierRetention(): array
    {
        return Capsule::select("
            SELECT 
                lm.tier,
                COUNT(DISTINCT lm.client_id) as total,
                COUNT(DISTINCT CASE WHEN c.status = 'Active' THEN lm.client_id END) as retained
            FROM tblloyalty_members lm
            LEFT JOIN tblclients c ON lm.client_id = c.id
            GROUP BY lm.tier
        ");
    }
    
    private function getRetentionTrend(int $months = 6): array
    {
        $trend = [];
        
        for ($i = $months - 1; $i >= 0; $i--) {
            $monthStart = date('Y-m-01', strtotime("-{$i} months"));
            $monthEnd = date('Y-m-t', strtotime("-{$i} months"));
            
            $totalClients = Capsule::table('tblclients')
                ->where('createdat', '<', $monthEnd)
                ->count();
            
            $churnedClients = Capsule::table('tblclients')
                ->where('status', 'Closed')
                ->whereBetween('updated_at', [$monthStart, $monthEnd])
                ->count();
            
            $retentionRate = $totalClients > 0 
                ? (($totalClients - $churnedClients) / $totalClients) * 100 
                : 0;
            
            $trend[] = [
                'month' => date('Y-M', strtotime("-{$i} months")),
                'retention_rate' => round($retentionRate, 2),
                'churned' => $churnedClients
            ];
        }
        
        return $trend;
    }
    
    public function generateOutput($data): string
    {
        $bronze = $silver = $gold = $platinum = 0;
        
        foreach ($data['loyalty_stats'] as $stat) {
            switch ($stat->tier) {
                case 'bronze': $bronze = $stat->members; break;
                case 'silver': $silver = $stat->members; break;
                case 'gold': $gold = $stat->members; break;
                case 'platinum': $platinum = $stat->members; break;
            }
        }
        
        $latestRetention = end($data['retention_trend']);
        
        return <<<HTML
<div class="retention-dashboard">
    <div class="loyalty-tiers">
        <div class="tier bronze"><span>{$bronze}</span>Bronze</div>
        <div class="tier silver"><span>{$silver}</span>Silver</div>
        <div class="tier gold"><span>{$gold}</span>Gold</div>
        <div class="tier platinum"><span>{$platinum}</span>Platinum</div>
    </div>
    <div class="retention-metrics">
        <span>Retention: {$latestRetention['retention_rate']}%</span>
        <span>Active Campaigns: {$data['active_campaigns']}</span>
    </div>
</div>
HTML;
    }
}
```

## Retention Program Summary

| Tier | Points | Discount | Benefits |
|------|--------|----------|----------|
| Bronze | 0-499 | 0% | Basic support |
| Silver | 500-1999 | 5% | Priority tickets |
| Gold | 2000-4999 | 10% | Priority support, early access |
| Platinum | 5000+ | 15% | Dedicated support, beta access |

## Best Practices

1. **Personalization**: Tailor communications based on customer history
2. **Multiple Touchpoints**: Use email, phone, in-app, and social
3. **Reward Loyalty**: Recognize and reward long-term customers
4. **Proactive Support**: Address issues before customers complain
5. **Measure NPS**: Track satisfaction through regular surveys
6. **Continuous Improvement**: Iterate based on feedback

## Common Pitfalls

- **One-Size-Fits-All**: Not segmenting retention efforts
- **Reactive Only**: Waiting for customers to complain
- **No Value Demonstration**: Not showing ROI of services
- **Ignoring Feedback**: Not acting on customer input
- **No Follow-through**: Initial outreach but no continued engagement

## Verification Checklist

- [ ] Retention scoring implemented
- [ ] Loyalty program configured
- [ ] Engagement programs in place
- [ ] Anniversary celebrations enabled
- [ ] Win-back campaigns active
- [ ] Dashboard widget deployed
- [ ] Program effectiveness tracked

## Related Documentation

- [WHMCS Churn Reduction](whmcs-churn-reduction.md)
- [WHMCS Client Segmentation](whmcs-client-segmentation.md)
- [WHMCS Revenue Optimization](whmcs-revenue-optimization.md)