# WHMCS Churn Reduction Workflow

## Purpose

Implement comprehensive churn reduction strategies for WHMCS to identify at-risk customers early, intervene proactively, and reduce customer attrition. This workflow covers churn prediction, prevention strategies, and retention programs.

## Prerequisites

- WHMCS v8.0+
- Admin access to WHMCS
- Customer lifecycle data
- Communication system (email/API)

## Workflow Steps

### Step 1: Churn Analysis Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                   Churn Reduction Framework                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Churn Indicators:                                               │
│  ┌─────────────────┐  ┌─────────────────┐                      │
│  │   Behavioral    │  │   Financial     │                      │
│  │  • Login drop   │  │  • Payment fail │                      │
│  │  • Support spike│  │  • Late pay     │                      │
│  │  • Usage decline│  │  • Credit issue │                      │
│  └─────────────────┘  └─────────────────┘                      │
│  ┌─────────────────┐  ┌─────────────────┐                      │
│  │   Service       │  │   Engagement    │                      │
│  │  • Downgrades   │  │  • No logins    │                      │
│  │  • Cancellations│  │  • No tickets   │                      │
│  │  • Expiry soon  │  │  • No purchases │                      │
│  └─────────────────┘  └─────────────────┘                      │
│                                                                  │
│  Churn Prevention Flow:                                          │
│                                                                  │
│  Identify ──▶ Score ──▶ Segment ──▶ Intervene ──▶ Recover     │
│     │          │           │           │            │         │
│     ▼          ▼           ▼           ▼            ▼         │
│  Monitor   Risk Level   Action Plan   Campaigns    Follow-up   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: Churn Prediction Model

```php
<?php
// /var/www/html/whmcs/includes/helpers/ChurnPredictor.php

namespace WHMCS\Helpers;

use Illuminate\Database\Capsule\Manager as Capsule;

class ChurnPredictor
{
    private $thresholds = [
        'login_declining' => 30,  // days since last login
        'support_spike' => 3,      // tickets in last 14 days
        'payment_failures' => 2,   // failed payments
        'usage_drop' => 50,        // percentage drop
    ];
    
    /**
     * Calculate churn risk score for client
     */
    public static function calculateChurnRisk(int $clientId): array
    {
        $riskFactors = [];
        $riskScore = 0;
        
        // Factor 1: Login activity
        $lastLogin = self::getLastLogin($clientId);
        if ($lastLogin) {
            $daysSinceLogin = (time() - strtotime($lastLogin)) / (24 * 3600);
            if ($daysSinceLogin > 90) {
                $riskFactors[] = ['factor' => 'inactive', 'weight' => 40, 'value' => $daysSinceLogin];
                $riskScore += 40;
            } elseif ($daysSinceLogin > 30) {
                $riskFactors[] = ['factor' => 'low_activity', 'weight' => 20, 'value' => $daysSinceLogin];
                $riskScore += 20;
            }
        }
        
        // Factor 2: Support tickets
        $recentTickets = self::getRecentTicketCount($clientId);
        if ($recentTickets >= $this->thresholds['support_spike']) {
            $riskFactors[] = ['factor' => 'support_spike', 'weight' => 15, 'value' => $recentTickets];
            $riskScore += 15;
        }
        
        // Factor 3: Payment history
        $paymentIssues = self::getPaymentIssues($clientId);
        if ($paymentIssues > 0) {
            $riskFactors[] = ['factor' => 'payment_issues', 'weight' => $paymentIssues * 10, 'value' => $paymentIssues];
            $riskScore += $paymentIssues * 10;
        }
        
        // Factor 4: Declining engagement
        $engagementTrend = self::getEngagementTrend($clientId);
        if ($engagementTrend < -50) {
            $riskFactors[] = ['factor' => 'declining_engagement', 'weight' => 25, 'value' => $engagementTrend];
            $riskScore += 25;
        }
        
        // Factor 5: Upcoming service expiry
        $expiringServices = self::getExpiringServices($clientId);
        if (count($expiringServices) > 0) {
            $riskFactors[] = ['factor' => 'expiring_services', 'weight' => 10, 'value' => count($expiringServices)];
            $riskScore += 10;
        }
        
        // Factor 6: Recent downgrade
        $recentDowngrades = self::getRecentDowngrades($clientId);
        if ($recentDowngrades > 0) {
            $riskFactors[] = ['factor' => 'recent_downgrade', 'weight' => 20, 'value' => $recentDowngrades];
            $riskScore += 20;
        }
        
        // Determine risk level
        $riskLevel = 'low';
        if ($riskScore >= 50) {
            $riskLevel = 'high';
        } elseif ($riskScore >= 25) {
            $riskLevel = 'medium';
        }
        
        return [
            'client_id' => $clientId,
            'risk_score' => min($riskScore, 100),
            'risk_level' => $riskLevel,
            'risk_factors' => $riskFactors,
            'recommended_actions' => self::getRecommendedActions($riskLevel, $riskFactors)
        ];
    }
    
    /**
     * Get last login time
     */
    private static function getLastLogin(int $clientId): ?string
    {
        return Capsule::table('tblactivitylog')
            ->where('userid', $clientId)
            ->whereIn('action', ['Login', 'Client Login'])
            ->max('date');
    }
    
    /**
     * Get recent ticket count
     */
    private static function getRecentTicketCount(int $clientId): int
    {
        return Capsule::table('tbltickets')
            ->where('userid', $clientId)
            ->where('created', '>=', date('Y-m-d', strtotime('-14 days')))
            ->count();
    }
    
    /**
     * Get payment issues count
     */
    private static function getPaymentIssues(int $clientId): int
    {
        return Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->where('status', 'Failed')
            ->where('date', '>=', date('Y-m-d', strtotime('-90 days')))
            ->count();
    }
    
    /**
     * Calculate engagement trend
     */
    private static function getEngagementTrend(int $clientId): float
    {
        $recentLogins = Capsule::table('tblactivitylog')
            ->where('userid', $clientId)
            ->where('date', '>=', date('Y-m-d', strtotime('-30 days')))
            ->count();
        
        $olderLogins = Capsule::table('tblactivitylog')
            ->where('userid', $clientId)
            ->whereBetween('date', [
                date('Y-m-d', strtotime('-60 days')),
                date('Y-m-d', strtotime('-30 days'))
            ])
            ->count();
        
        if ($olderLogins === 0) {
            return $recentLogins > 0 ? 0 : -100;
        }
        
        return (($recentLogins - $olderLogins) / $olderLogins) * 100;
    }
    
    /**
     * Get expiring services
     */
    private static function getExpiringServices(int $clientId): array
    {
        return Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->whereIn('domainstatus', ['Active', 'Suspended'])
            ->where('nextduedate', '<=', date('Y-m-d', strtotime('+14 days')))
            ->get()
            ->toArray();
    }
    
    /**
     * Get recent downgrades
     */
    private static function getRecentDowngrades(int $clientId): int
    {
        return Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->where('domainstatus', 'Active')
            ->where('last_update', '>=', date('Y-m-d', strtotime('-30 days')))
            ->count();
    }
    
    /**
     * Get recommended actions based on risk level
     */
    private static function getRecommendedActions(string $riskLevel, array $riskFactors): array
    {
        $actions = [];
        
        foreach ($riskFactors as $factor) {
            switch ($factor['factor']) {
                case 'inactive':
                    $actions[] = ['action' => 'winback_email', 'priority' => 'high', 'message' => 'Send re-engagement email'];
                    break;
                case 'support_spike':
                    $actions[] = ['action' => 'proactive_support', 'priority' => 'high', 'message' => 'Proactive outreach to resolve issues'];
                    break;
                case 'payment_issues':
                    $actions[] = ['action' => 'payment_plan', 'priority' => 'high', 'message' => 'Offer payment plan or extension'];
                    break;
                case 'declining_engagement':
                    $actions[] = ['action' => 'usage_campaign', 'priority' => 'medium', 'message' => 'Send usage tips and feature highlights'];
                    break;
                case 'expiring_services':
                    $actions[] = ['action' => 'renewal_offer', 'priority' => 'medium', 'message' => 'Send renewal reminder with incentive'];
                    break;
                case 'recent_downgrade':
                    $actions[] = ['action' => 'follow_up', 'priority' => 'medium', 'message' => 'Follow up to understand needs'];
                    break;
            }
        }
        
        return $actions;
    }
}
```

### Step 3: Automated Churn Detection

```php
<?php
// /var/www/html/whmcs/includes/hooks/churn_detection.php

add_hook('DailyCronJob', 1, function() {
    logActivity('Starting churn risk analysis');
    
    // Get all active clients
    $clients = \WHMCS\User\Client::where('status', 'Active')->get();
    
    $highRisk = 0;
    $mediumRisk = 0;
    $lowRisk = 0;
    
    foreach ($clients as $client) {
        $riskAssessment = \WHMCS\Helpers\ChurnPredictor::calculateChurnRisk($client->id);
        
        // Store risk assessment
        Capsule::table('tblchurn_risk')->updateOrInsert(
            ['client_id' => $client->id],
            [
                'risk_score' => $riskAssessment['risk_score'],
                'risk_level' => $riskAssessment['risk_level'],
                'risk_factors' => json_encode($riskAssessment['risk_factors']),
                'recommended_actions' => json_encode($riskAssessment['recommended_actions']),
                'assessed_at' => date('Y-m-d H:i:s')
            ]
        );
        
        // Track counts
        switch ($riskAssessment['risk_level']) {
            case 'high':
                $highRisk++;
                self::triggerHighRiskInterventions($client->id, $riskAssessment);
                break;
            case 'medium':
                $mediumRisk++;
                self::triggerMediumRiskInterventions($client->id, $riskAssessment);
                break;
            case 'low':
                $lowRisk++;
                break;
        }
    }
    
    logActivity("Churn analysis complete: {$highRisk} high, {$mediumRisk} medium, {$lowRisk} low risk");
});

/**
 * Trigger interventions for high-risk clients
 */
function triggerHighRiskInterventions(int $clientId, array $riskAssessment): void
{
    // Create support ticket flag
    Capsule::table('tblclient_flags')->insert([
        'client_id' => $clientId,
        'flag' => 'churn_risk',
        'notes' => 'High risk client - review recommended',
        'created_by' => 'system',
        'created_at' => date('Y-m-d H:i:s')
    ]);
    
    // Queue email notification
    Capsule::table('tblchurn_campaign_queue')->insert([
        'client_id' => $clientId,
        'campaign_type' => 'high_risk_winback',
        'priority' => 'high',
        'scheduled_for' => date('Y-m-d H:i:s', strtotime('+1 day')),
        'status' => 'pending'
    ]);
}

/**
 * Trigger interventions for medium-risk clients
 */
function triggerMediumRiskInterventions(int $clientId, array $riskAssessment): void
{
    // Queue engagement campaign
    Capsule::table('tblchurn_campaign_queue')->insert([
        'client_id' => $clientId,
        'campaign_type' => 'medium_risk_engagement',
        'priority' => 'medium',
        'scheduled_for' => date('Y-m-d H:i:s', strtotime('+3 days')),
        'status' => 'pending'
    ]);
}
```

### Step 4: Churn Risk Storage Schema

```sql
-- Create churn risk tracking table
CREATE TABLE IF NOT EXISTS `tblchurn_risk` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT UNSIGNED NOT NULL,
    `risk_score` DECIMAL(5,2) DEFAULT 0,
    `risk_level` ENUM('low', 'medium', 'high') DEFAULT 'low',
    `risk_factors` JSON,
    `recommended_actions` JSON,
    `assessed_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    `intervention_sent` TINYINT(1) DEFAULT 0,
    `intervention_sent_at` TIMESTAMP NULL,
    UNIQUE KEY `idx_client_id` (`client_id`),
    INDEX `idx_risk_level` (`risk_level`),
    INDEX `idx_assessed_at` (`assessed_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Create churn campaign queue table
CREATE TABLE IF NOT EXISTS `tblchurn_campaign_queue` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT UNSIGNED NOT NULL,
    `campaign_type` VARCHAR(50) NOT NULL,
    `priority` ENUM('low', 'medium', 'high') DEFAULT 'medium',
    `scheduled_for` DATETIME,
    `status` ENUM('pending', 'sent', 'failed', 'cancelled') DEFAULT 'pending',
    `sent_at` TIMESTAMP NULL,
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_status_scheduled` (`status`, `scheduled_for`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Create churn recovery tracking
CREATE TABLE IF NOT EXISTS `tblchurn_recovery` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT UNSIGNED NOT NULL,
    `risk_score_before` DECIMAL(5,2),
    `risk_score_after` DECIMAL(5,2),
    `intervention_type` VARCHAR(50),
    `recovered` TINYINT(1) DEFAULT 0,
    `recovered_at` TIMESTAMP NULL,
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_client_id` (`client_id`),
    INDEX `idx_recovered` (`recovered`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Step 5: Churn Prevention Campaigns

```php
<?php
// /var/www/html/whmcs/includes/helpers/ChurnCampaignManager.php

namespace WHMCS\Helpers;

class ChurnCampaignManager
{
    /**
     * Process pending campaigns
     */
    public static function processPendingCampaigns(): void
    {
        $pending = Capsule::table('tblchurn_campaign_queue')
            ->where('status', 'pending')
            ->where('scheduled_for', '<=', date('Y-m-d H:i:s'))
            ->get();
        
        foreach ($pending as $campaign) {
            self::sendCampaign($campaign);
        }
    }
    
    /**
     * Send campaign to client
     */
    private static function sendCampaign(object $campaign): void
    {
        $client = \WHMCS\User\Client::find($campaign->client_id);
        
        if (!$client) {
            Capsule::table('tblchurn_campaign_queue')
                ->where('id', $campaign->id)
                ->update(['status' => 'failed']);
            return;
        }
        
        $emailTemplate = self::getEmailTemplate($campaign->campaign_type);
        
        // Send email
        try {
            sendEmail($emailTemplate, $client->id, [
                'client_name' => $client->firstname . ' ' . $client->lastname
            ]);
            
            Capsule::table('tblchurn_campaign_queue')
                ->where('id', $campaign->id)
                ->update([
                    'status' => 'sent',
                    'sent_at' => date('Y-m-d H:i:s')
                ]);
        } catch (\Exception $e) {
            Capsule::table('tblchurn_campaign_queue')
                ->where('id', $campaign->id)
                ->update(['status' => 'failed']);
            
            logActivity('Churn campaign failed: ' . $e->getMessage());
        }
    }
    
    /**
     * Get email template for campaign type
     */
    private static function getEmailTemplate(string $campaignType): string
    {
        $templates = [
            'high_risk_winback' => 'ChurnWinbackHighRisk',
            'medium_risk_engagement' => 'ChurnEngagementMedium',
            'reengagement' => 'ChurnReengagement',
            'renewal_reminder' => 'ChurnRenewalReminder'
        ];
        
        return $templates[$campaignType] ?? 'GeneralReminder';
    }
    
    /**
     * Track intervention effectiveness
     */
    public static function trackIntervention(int $clientId, string $interventionType): void
    {
        $riskBefore = Capsule::table('tblchurn_risk')
            ->where('client_id', $clientId)
            ->first();
        
        Capsule::table('tblchurn_recovery')->insert([
            'client_id' => $clientId,
            'risk_score_before' => $riskBefore->risk_score ?? 0,
            'intervention_type' => $interventionType
        ]);
    }
    
    /**
     * Check if intervention worked
     */
    public static function checkInterventionEffectiveness(int $clientId): bool
    {
        $before = Capsule::table('tblchurn_recovery')
            ->where('client_id', $clientId)
            ->orderBy('created_at', 'desc')
            ->first();
        
        if (!$before) {
            return false;
        }
        
        $current = Capsule::table('tblchurn_risk')
            ->where('client_id', $clientId)
            ->first();
        
        if (!$current) {
            return false;
        }
        
        $recovered = $current->risk_score < $before->risk_score_before - 20;
        
        Capsule::table('tblchurn_recovery')
            ->where('id', $before->id)
            ->update([
                'risk_score_after' => $current->risk_score,
                'recovered' => $recovered ? 1 : 0,
                'recovered_at' => $recovered ? date('Y-m-d H:i:s') : null
            ]);
        
        return $recovered;
    }
}
```

### Step 6: Churn Dashboard Widget

```php
<?php
// /var/www/html/whmcs/modules/widgets/ChurnDashboard.php

namespace WHMCS\Module\Widget;

class ChurnDashboard extends \WHMCS\Module\AbstractWidget
{
    protected $title = 'Churn Risk Overview';
    protected $author = 'System';
    protected $type = 'dashboard';
    
    public function getData(): array
    {
        $riskDistribution = Capsule::select("
            SELECT 
                risk_level,
                COUNT(*) as count
            FROM tblchurn_risk
            GROUP BY risk_level
        ");
        
        $highRiskClients = Capsule::table('tblchurn_risk')
            ->where('risk_level', 'high')
            ->orderBy('risk_score', 'desc')
            ->limit(10)
            ->get();
        
        $recoveryRate = $this->calculateRecoveryRate();
        
        return [
            'risk_distribution' => $riskDistribution,
            'high_risk_clients' => $highRiskClients,
            'recovery_rate' => $recoveryRate,
            'total_at_risk' => array_sum(array_column($riskDistribution, 'count'))
        ];
    }
    
    private function calculateRecoveryRate(): float
    {
        $total = Capsule::table('tblchurn_recovery')->count();
        $recovered = Capsule::table('tblchurn_recovery')->where('recovered', 1)->count();
        
        return $total > 0 ? round(($recovered / $total) * 100, 2) : 0;
    }
    
    public function generateOutput($data): string
    {
        $highRisk = 0;
        $mediumRisk = 0;
        $lowRisk = 0;
        
        foreach ($data['risk_distribution'] as $dist) {
            switch ($dist->risk_level) {
                case 'high':
                    $highRisk = $dist->count;
                    break;
                case 'medium':
                    $mediumRisk = $dist->count;
                    break;
                case 'low':
                    $lowRisk = $dist->count;
                    break;
            }
        }
        
        return <<<HTML
<div class="churn-dashboard">
    <div class="risk-summary">
        <div class="risk-high">
            <span class="count">{$highRisk}</span>
            <span class="label">High Risk</span>
        </div>
        <div class="risk-medium">
            <span class="count">{$mediumRisk}</span>
            <span class="label">Medium Risk</span>
        </div>
        <div class="risk-low">
            <span class="count">{$lowRisk}</span>
            <span class="label">Low Risk</span>
        </div>
    </div>
    <div class="recovery-rate">
        <span>Recovery Rate: {$data['recovery_rate']}%</span>
    </div>
</div>
HTML;
    }
}
```

## Churn Metrics Summary

| Metric | Formula | Target |
|--------|---------|--------|
| Monthly Churn Rate | (Churned / Total) * 100 | <3% |
| Net Revenue Churn | (Lost MRR - Expansion MRR) / Start MRR | <2% |
| Churn Risk Score | Weighted factor sum | Monitor |
| Recovery Rate | (Recovered / At-Risk) * 100 | >60% |
| Time to Churn | Average days to churn | >180 days |

## Best Practices

1. **Early Detection**: Monitor engagement metrics for early warning signs
2. **Proactive Outreach**: Reach out before customer decides to leave
3. **Personalize Communication**: Tailor messages based on risk factors
4. **Multiple Touchpoints**: Use email, phone, and in-app messaging
5. **Measure Effectiveness**: Track recovery rate and adjust strategies
6. **Address Root Causes**: Understand why customers are churning

## Common Pitfalls

- **Reactive Approach**: Waiting until customer is about to leave
- **Generic Messaging**: Not personalizing based on customer situation
- **Single Channel**: Using only one communication method
- **No Follow-up**: Not continuing to engage after initial outreach
- **Ignoring Feedback**: Not addressing the reasons for churn

## Verification Checklist

- [ ] Churn prediction model implemented
- [ ] Risk scoring configured
- [ ] Automated detection in place
- [ ] Intervention campaigns created
- [ ] Recovery tracking enabled
- [ ] Dashboard widget deployed
- [ ] Campaign effectiveness monitored

## Related Documentation

- [WHMCS Customer Retention](whmcs-customer-retention.md)
- [WHMCS Client Segmentation](whmcs-client-segmentation.md)
- [WHMCS Revenue Optimization](whmcs-revenue-optimization.md)