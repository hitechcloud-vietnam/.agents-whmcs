# WHMCS Client Segmentation Workflow

## Purpose

Implement comprehensive client segmentation for WHMCS to enable targeted marketing, personalized services, and improved customer experience. This workflow covers segmentation criteria, implementation, and automated segmentation management.

## Prerequisites

- WHMCS v8.0+
- Admin access to WHMCS
- Understanding of business metrics
- Marketing automation integration (optional)

## Workflow Steps

### Step 1: Segmentation Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHMCS Client Segmentation Model                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Behavioral │  │   Demographic│  │   Engagement │          │
│  │  Segments    │  │  Segments    │  │  Segments    │          │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
│  │ • Purchasers │  │ • By Revenue │  │ • Active     │          │
│  │ • Trial Users│  │ • By Industry│  │ • At-Risk    │          │
│  │ • Churned    │  │ • By Size    │  │ • Lapsed     │          │
│  │ • Upsellers  │  │ • By Region  │  │ • Champions  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  Segment Criteria Scoring:                                        │
│                                                                  │
│  Revenue Score = (Monthly * 1) + (Annual * 12) + (One-time * 0.5)│
│  Engagement Score = (Logins * 0.3) + (Support Tickets * 0.2)    │
│                   + (Upgrades * 0.5)                             │
│  Loyalty Score = (Years * 10) + (Services * 2) - (Churns * 5)   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: Segment Definition Implementation

```php
<?php
// /var/www/html/whmcs/includes/helpers/ClientSegmentation.php

namespace WHMCS\Helpers;

use Illuminate\Database\Capsule\Manager as Capsule;
use WHMCS\User\Client;

class ClientSegmentation
{
    /**
     * Calculate client segment scores
     */
    public static function calculateSegmentScores(int $clientId): array
    {
        $client = Client::find($clientId);
        
        if (!$client) {
            return [];
        }
        
        return [
            'revenue_score' => self::calculateRevenueScore($clientId),
            'engagement_score' => self::calculateEngagementScore($clientId),
            'loyalty_score' => self::calculateLoyaltyScore($clientId),
            'risk_score' => self::calculateRiskScore($clientId),
            'lifetime_value' => self::calculateLifetimeValue($clientId),
        ];
    }
    
    /**
     * Calculate revenue score
     */
    private static function calculateRevenueScore(int $clientId): float
    {
        $stats = Capsule::select("
            SELECT 
                COALESCE(SUM(CASE WHEN i.status = 'Paid' THEN i.total ELSE 0 END), 0) as total_revenue,
                COUNT(DISTINCT i.id) as invoice_count,
                MAX(i.datepaid) as last_payment
            FROM tblinvoices i
            WHERE i.userid = ?
        ", [$clientId])[0];
        
        // Weight recent payments more heavily
        $monthsSincePayment = $stats->last_payment 
            ? (time() - strtotime($stats->last_payment)) / (30 * 24 * 3600)
            : 999;
        
        $recencyWeight = max(0, 1 - ($monthsSincePayment / 12));
        
        return round($stats->total_revenue * (0.5 + $recencyWeight * 0.5), 2);
    }
    
    /**
     * Calculate engagement score
     */
    private static function calculateEngagementScore(int $clientId): float
    {
        // Login frequency
        $logins = Capsule::table('tblactivitylog')
            ->where('userid', $clientId)
            ->where('date', '>=', date('Y-m-d', strtotime('-90 days')))
            ->count();
        
        // Support ticket activity
        $tickets = Capsule::table('tbltickets')
            ->where('userid', $clientId)
            ->where('created', '>=', date('Y-m-d', strtotime('-90 days')))
            ->count();
        
        // Service upgrades
        $upgrades = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->where('nextduedate', '>', date('Y-m-d'))
            ->count();
        
        // Calculate weighted score
        return round(
            ($logins * 0.3) + 
            ($tickets * 0.2) + 
            ($upgrades * 0.5),
            2
        );
    }
    
    /**
     * Calculate loyalty score
     */
    private static function calculateLoyaltyScore(int $clientId): float
    {
        $client = Client::find($clientId);
        
        // Years as customer
        $yearsActive = (time() - strtotime($client->createdat)) / (365 * 24 * 3600);
        
        // Active services
        $activeServices = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->where('domainstatus', 'Active')
            ->count();
        
        // Previous cancellations (simplified)
        $churnCount = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->where('domainstatus', 'Terminated')
            ->count();
        
        return round(
            ($yearsActive * 10) + 
            ($activeServices * 2) - 
            ($churnCount * 5),
            2
        );
    }
    
    /**
     * Calculate risk score (higher = more likely to churn)
     */
    private static function calculateRiskScore(int $clientId): float
    {
        $riskFactors = [];
        
        // No recent login
        $lastLogin = Capsule::table('tblactivitylog')
            ->where('userid', $clientId)
            ->whereIn('action', ['Login', 'Client Login'])
            ->max('date');
        
        if ($lastLogin && (time() - strtotime($lastLogin)) > 90 * 24 * 3600) {
            $riskFactors[] = 20; // No login in 90 days
        }
        
        // Payment issues
        $failedPayments = Capsule::table('tblinvoiceitems')
            ->join('tblinvoices', 'tblinvoiceitems.invoiceid', '=', 'tblinvoices.id')
            ->where('tblinvoices.userid', $clientId)
            ->where('tblinvoices.status', 'Failed')
            ->count();
        
        if ($failedPayments > 2) {
            $riskFactors[] = 30; // Multiple failed payments
        }
        
        // Declining engagement
        $recentLogins = Capsule::table('tblactivitylog')
            ->where('userid', $clientId)
            ->where('date', '>=', date('Y-m-d', strtotime('-30 days')))
            ->count();
        
        $olderLogins = Capsule::table('tblactivitylog')
            ->where('userid', $clientId)
            ->whereBetween('date', [
                date('Y-m-d', strtotime('-90 days')),
                date('Y-m-d', strtotime('-30 days'))
            ])
            ->count();
        
        if ($recentLogins < $olderLogins * 0.5) {
            $riskFactors[] = 25; // Declining engagement
        }
        
        return array_sum($riskFactors);
    }
    
    /**
     * Calculate lifetime value
     */
    private static function calculateLifetimeValue(int $clientId): float
    {
        $totalRevenue = Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->where('status', 'Paid')
            ->sum('total');
        
        $yearsActive = max(1, (time() - strtotime(
            Capsule::table('tblclients')->where('id', $clientId)->value('createdat')
        )) / (365 * 24 * 3600));
        
        // Average monthly revenue
        $avgMonthlyRevenue = $totalRevenue / $yearsActive / 12;
        
        // Estimated remaining years (simplified)
        $estimatedYearsRemaining = max(1, 3 - ($yearsActive * 0.2));
        
        return round($avgMonthlyRevenue * 12 * $estimatedYearsRemaining, 2);
    }
}
```

### Step 3: Automated Segment Assignment

```php
<?php
// /var/www/html/whmcs/includes/hooks/segment_assignment.php

add_hook('DailyCronJob', 1, function() {
    logActivity('Starting client segment recalculation');
    
    // Get all active clients
    $clients = \WHMCS\User\Client::where('status', 'Active')->get();
    
    foreach ($clients as $client) {
        $segments = \WHMCS\Helpers\ClientSegmentation::calculateSegmentScores($client->id);
        
        // Store segment data
        Capsule::table('tblclient_segments')->updateOrInsert(
            ['client_id' => $client->id],
            [
                'revenue_score' => $segments['revenue_score'],
                'engagement_score' => $segments['engagement_score'],
                'loyalty_score' => $segments['loyalty_score'],
                'risk_score' => $segments['risk_score'],
                'lifetime_value' => $segments['lifetime_value'],
                'updated_at' => date('Y-m-d H:i:s')
            ]
        );
        
        // Assign segment categories
        self::assignSegmentCategory($client->id, $segments);
    }
    
    logActivity('Client segment recalculation completed for ' . count($clients) . ' clients');
});

/**
 * Assign client to segment category based on scores
 */
function assignSegmentCategory(int $clientId, array $scores): void
{
    $segment = 'standard'; // Default
    
    // VIP: High revenue, high engagement, high loyalty
    if ($scores['revenue_score'] > 5000 && $scores['loyalty_score'] > 30) {
        $segment = 'vip';
    }
    // At-Risk: High risk score, low engagement
    elseif ($scores['risk_score'] > 30 || $scores['engagement_score'] < 5) {
        $segment = 'at_risk';
    }
    // High Potential: Good engagement, low revenue (upsell opportunity)
    elseif ($scores['engagement_score'] > 20 && $scores['revenue_score'] < 1000) {
        $segment = 'high_potential';
    }
    // Champions: High loyalty, high engagement
    elseif ($scores['loyalty_score'] > 25 && $scores['engagement_score'] > 15) {
        $segment = 'champion';
    }
    // Dormant: Low engagement, low activity
    elseif ($scores['engagement_score'] < 2) {
        $segment = 'dormant';
    }
    
    Capsule::table('tblclient_segments')
        ->where('client_id', $clientId)
        ->update(['segment_category' => $segment]);
}
```

### Step 4: Segment Storage Schema

```sql
-- Create client segments table
CREATE TABLE IF NOT EXISTS `tblclient_segments` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT UNSIGNED NOT NULL,
    `revenue_score` DECIMAL(10,2) DEFAULT 0,
    `engagement_score` DECIMAL(10,2) DEFAULT 0,
    `loyalty_score` DECIMAL(10,2) DEFAULT 0,
    `risk_score` DECIMAL(10,2) DEFAULT 0,
    `lifetime_value` DECIMAL(10,2) DEFAULT 0,
    `segment_category` VARCHAR(50) DEFAULT 'standard',
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    `updated_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY `idx_client_id` (`client_id`),
    INDEX `idx_segment_category` (`segment_category`),
    INDEX `idx_revenue_score` (`revenue_score`),
    INDEX `idx_risk_score` (`risk_score`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Create segment definitions table
CREATE TABLE IF NOT EXISTS `tblsegment_definitions` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `segment_name` VARCHAR(50) NOT NULL,
    `segment_description` TEXT,
    `revenue_min` DECIMAL(10,2) DEFAULT 0,
    `revenue_max` DECIMAL(10,2) DEFAULT NULL,
    `engagement_min` DECIMAL(10,2) DEFAULT 0,
    `risk_max` DECIMAL(10,2) DEFAULT 100,
    `is_active` TINYINT(1) DEFAULT 1,
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Insert default segment definitions
INSERT INTO `tblsegment_definitions` (`segment_name`, `segment_description`, `revenue_min`, `revenue_max`, `engagement_min`, `risk_max`) VALUES
('vip', 'High-value customers with strong loyalty', 5000, NULL, 20, 10),
('champion', 'Loyal customers with high engagement', 0, NULL, 15, 20),
('high_potential', 'Engaged customers with upsell opportunity', 0, 1000, 20, 30),
('at_risk', 'Customers showing signs of potential churn', 0, NULL, 0, 30),
('dormant', 'Inactive customers requiring re-engagement', 0, NULL, 0, 100),
('standard', 'Regular customers', 0, NULL, 2, NULL);
```

### Step 5: Segment-Based Service Recommendations

```php
<?php
// /var/www/html/whmcs/includes/helpers/SegmentRecommendations.php

namespace WHMCS\Helpers;

class SegmentRecommendations
{
    /**
     * Get recommendations based on segment
     */
    public static function getRecommendations(int $clientId): array
    {
        $segment = Capsule::table('tblclient_segments')
            ->where('client_id', $clientId)
            ->first();
        
        if (!$segment) {
            return [];
        }
        
        return self::recommendationsForSegment($segment->segment_category, $segment);
    }
    
    /**
     * Generate recommendations for segment
     */
    private static function recommendationsForSegment(string $category, object $scores): array
    {
        $recommendations = [];
        
        switch ($category) {
            case 'vip':
                $recommendations = [
                    ['type' => 'upgrade', 'priority' => 'high', 'message' => 'Consider premium support package'],
                    ['type' => 'referral', 'priority' => 'medium', 'message' => 'Invite to referral program'],
                    ['type' => 'retention', 'priority' => 'low', 'message' => 'Provide exclusive benefits']
                ];
                break;
                
            case 'at_risk':
                $recommendations = [
                    ['type' => 'winback', 'priority' => 'high', 'message' => 'Send re-engagement offer'],
                    ['type' => 'support', 'priority' => 'high', 'message' => 'Proactive outreach'],
                    ['type' => 'discount', 'priority' => 'medium', 'message' => 'Consider loyalty discount']
                ];
                break;
                
            case 'high_potential':
                $recommendations = [
                    ['type' => 'upsell', 'priority' => 'high', 'message' => 'Suggest premium products'],
                    ['type' => 'education', 'priority' => 'medium', 'message' => 'Provide product tutorials'],
                    ['type' => 'success', 'priority' => 'medium', 'message' => 'Onboarding assistance']
                ];
                break;
                
            case 'champion':
                $recommendations = [
                    ['type' => 'referral', 'priority' => 'high', 'message' => 'Request review/testimonial'],
                    ['type' => 'community', 'priority' => 'medium', 'message' => 'Invite to beta program'],
                    ['type' => 'upsell', 'priority' => 'low', 'message' => 'Cross-sell related products']
                ];
                break;
                
            case 'dormant':
                $recommendations = [
                    ['type' => 'winback', 'priority' => 'high', 'message' => 'Send reactivation campaign'],
                    ['type' => 'survey', 'priority' => 'medium', 'message' => 'Understand disengagement reason'],
                    ['type' => 'incentive', 'priority' => 'high', 'message' => 'Provide special offer to return']
                ];
                break;
        }
        
        // Add personalized recommendations based on scores
        if ($scores->lifetime_value > 10000 && $category !== 'vip') {
            $recommendations[] = [
                'type' => 'upgrade_segment',
                'priority' => 'high',
                'message' => 'Customer qualifies for VIP tier'
            ];
        }
        
        return $recommendations;
    }
}
```

### Step 6: Segment Analytics Dashboard

```php
<?php
// /var/www/html/whmcs/modules/widgets/SegmentDashboard.php

namespace WHMCS\Module\Widget;

class SegmentDashboard extends \WHMCS\Module\AbstractWidget
{
    protected $title = 'Client Segments';
    protected $author = 'System';
    protected $type = 'dashboard';
    
    public function getData(): array
    {
        $segments = Capsule::select("
            SELECT 
                segment_category,
                COUNT(*) as client_count,
                AVG(revenue_score) as avg_revenue,
                AVG(lifetime_value) as avg_ltv,
                AVG(risk_score) as avg_risk
            FROM tblclient_segments
            GROUP BY segment_category
            ORDER BY avg_ltv DESC
        ");
        
        // Monthly segment changes
        $changes = Capsule::select("
            SELECT 
                segment_category,
                COUNT(*) as new_members
            FROM tblclient_segments
            WHERE updated_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
            GROUP BY segment_category
        ");
        
        return [
            'segments' => $segments,
            'changes' => $changes,
            'total_clients' => array_sum(array_column($segments, 'client_count'))
        ];
    }
    
    public function generateOutput($data): string
    {
        $html = '<div class="segment-dashboard">';
        
        foreach ($data['segments'] as $segment) {
            $color = $this->getSegmentColor($segment->segment_category);
            $html .= "
                <div class='segment-card' style='border-left: 4px solid {$color}'>
                    <h4>{$segment->segment_category}</h4>
                    <div class='stat'>
                        <span class='count'>{$segment->client_count}</span>
                        <span class='label'>clients</span>
                    </div>
                    <div class='stat'>
                        <span class='value'>\${$segment->avg_ltv}</span>
                        <span class='label'>avg LTV</span>
                    </div>
                </div>
            ";
        }
        
        return $html . '</div>';
    }
    
    private function getSegmentColor(string $category): string
    {
        $colors = [
            'vip' => '#gold',
            'champion' => '#28a745',
            'high_potential' => '#17a2b8',
            'at_risk' => '#dc3545',
            'dormant' => '#6c757d',
            'standard' => '#007bff'
        ];
        return $colors[$category] ?? '#007bff';
    }
}
```

## Segment Categories Summary

| Segment | Criteria | Count | Avg LTV | Strategy |
|---------|----------|-------|---------|----------|
| VIP | Revenue > $5000, Loyalty > 30 | Variable | $15,000+ | Retention, Referral |
| Champion | Engagement > 15, Loyalty > 25 | Variable | $8,000+ | Advocacy, Beta |
| High Potential | Engagement > 20, Revenue < $1000 | Variable | $5,000+ | Upsell, Education |
| At Risk | Risk Score > 30 | Variable | $2,000+ | Winback, Support |
| Dormant | Engagement < 2 | Variable | $500+ | Reactivation |
| Standard | Default segment | Variable | $1,500+ | General Marketing |

## Best Practices

1. **Multi-Dimensional**: Use multiple criteria for segment assignment
2. **Dynamic Updates**: Recalculate segments regularly
3. **Clear Definitions**: Document segment criteria
4. **Actionable Segments**: Each segment should have specific actions
5. **Size Balance**: Avoid segments with very few or very many members
6. **Regular Review**: Re-evaluate segment definitions quarterly
7. **Automated Triggers**: Trigger actions based on segment changes

## Common Pitfalls

- **Overlapping Segments**: Clients in multiple segments
- **Static Segments**: Not updating as customer behavior changes
- **Too Many Segments**: Overcomplicating segmentation
- **No Action**: Creating segments without corresponding actions
- **Manual Assignment**: Not automating segment updates

## Verification Checklist

- [ ] Segment criteria defined
- [ ] Scoring algorithms implemented
- [ ] Automated segment assignment in place
- [ ] Segment storage schema created
- [ ] Recommendations engine built
- [ ] Dashboard widget created
- [ ] Segment-based campaigns configured

## Related Documentation

- [WHMCS Customer Retention](whmcs-customer-retention.md)
- [WHMCS Churn Reduction](whmcs-churn-reduction.md)
- [WHMCS Revenue Optimization](whmcs-revenue-optimization.md)