# WHMCS Cost Management Workflow

## Purpose

Implement comprehensive cost management for WHMCS infrastructure to optimize spending, reduce waste, and improve resource efficiency. This workflow covers cloud cost optimization, server resource management, and operational expense tracking.

## Prerequisites

- WHMCS v8.0+
- Cloud infrastructure (AWS, GCP, Azure, or similar)
- SSH access to servers
- Admin access to WHMCS
- Cost monitoring tools

## Workflow Steps

### Step 1: Cost Analysis Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                   Cloud Cost Management Framework                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Cost Categories:                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Compute    │  │   Storage    │  │   Network    │          │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
│  │ • Servers    │  │ • Databases  │  │ • Bandwidth  │          │
│  │ • Containers │  │ • Backups    │  │ • CDN        │          │
│  │ • Functions  │  │ • Snapshots  │  │ • Load Bal.  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  Optimization Levers:                                           │
│  ┌─────────────────┐  ┌─────────────────┐                       │
│  │   Right-Sizing  │  │   Auto-Scaling  │                       │
│  │  Match resources│  │   Scale demand  │                       │
│  │  to actual needs│  │   only          │                       │
│  └─────────────────┘  └─────────────────┘                       │
│  ┌─────────────────┐  ┌─────────────────┐                       │
│  │   Reserved Cap.  │  │   Spot/Preempt  │                       │
│  │   Save 30-60%   │  │   Save 70-90%   │                       │
│  └─────────────────┘  └─────────────────┘                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: Cost Tracking Implementation

```php
<?php
// /var/www/html/whmcs/includes/helpers/CostTracker.php

namespace WHMCS\Helpers;

use Illuminate\Database\Capsule\Manager as Capsule;

class CostTracker
{
    /**
     * Track infrastructure costs
     */
    public static function trackInfrastructureCost(string $period = '30 days'): array
    {
        return [
            'compute' => self::getComputeCosts($period),
            'storage' => self::getStorageCosts($period),
            'network' => self::getNetworkCosts($period),
            'third_party' => self::getThirdPartyCosts($period),
            'total' => self::calculateTotalCost($period),
            'by_service' => self::getCostsByService()
        ];
    }

    /**
     * Get compute costs
     */
    private static function getComputeCosts(string $period): array
    {
        $startDate = date('Y-m-d', strtotime("-{$period}"));

        // Server costs from provisioning records
        $serverCosts = Capsule::select("
            SELECT 
                s.name as server_name,
                s.ipaddress,
                SUM(h.amount) as monthly_cost,
                COUNT(h.id) as services
            FROM tblhosting h
            INNER JOIN tblservers s ON h.server = s.id
            WHERE h.domainstatus = 'Active'
            GROUP BY s.id, s.name, s.ipaddress
        ");

        // Calculate total compute cost
        $totalCompute = array_sum(array_column($serverCosts, 'monthly_cost'));

        return [
            'total' => round($totalCompute, 2),
            'by_server' => $serverCosts,
            'avg_per_service' => count($serverCosts) > 0 
                ? round($totalCompute / array_sum(array_column($serverCosts, 'services')), 2) 
                : 0
        ];
    }

    /**
     * Get storage costs
     */
    private static function getStorageCosts(string $period): array
    {
        // Estimate based on service data
        $totalStorageGB = Capsule::select("
            SELECT 
                SUM(COALESCE(diskUsage, 0)) as total_disk_gb,
                COUNT(*) as services
            FROM tblhosting
            WHERE domainstatus = 'Active'
        ")[0] ?? ['total_disk_gb' => 0, 'services' => 0];

        // Storage cost estimation ($0.10/GB standard, $0.025/GB for S3-like)
        $storageCostPerGB = 0.10;
        $monthlyStorageCost = ($totalStorageGB->total_disk_gb / 1024) * $storageCostPerGB;

        // Backup storage
        $backupCost = $monthlyStorageCost * 0.5; // Backups typically 50% of primary

        return [
            'primary_storage' => round($monthlyStorageCost, 2),
            'backup_storage' => round($backupCost, 2),
            'total' => round($monthlyStorageCost + $backupCost, 2),
            'total_gb' => round($totalStorageGB->total_disk_gb / 1024, 2)
        ];
    }

    /**
     * Get network costs
     */
    private static function getNetworkCosts(string $period): array
    {
        // Bandwidth estimation
        $activeServices = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count();

        // Average bandwidth per service (1GB assumed)
        $avgBandwidthGB = $activeServices;
        $bandwidthCostPerGB = 0.09;
        $bandwidthCost = $avgBandwidthGB * $bandwidthCostPerGB;

        // CDN costs (if applicable)
        $cdnCost = $bandwidthCost * 0.3;

        return [
            'bandwidth' => round($bandwidthCost, 2),
            'cdn' => round($cdnCost, 2),
            'total' => round($bandwidthCost + $cdnCost, 2),
            'estimated_gb' => $avgBandwidthGB
        ];
    }

    /**
     * Get third-party costs
     */
    private static function getThirdPartyCosts(string $period): array
    {
        $startDate = date('Y-m-d', strtotime("-{$period}"));

        // Domain registration costs
        $domainCosts = Capsule::select("
            SELECT 
                SUM(d.recurring_cost) as annual_cost,
                COUNT(*) as domains
            FROM tbldomains d
            WHERE d.status NOT IN ('Expired', 'Transferred')
        ")[0] ?? ['annual_cost' => 0, 'domains' => 0];

        // Payment gateway fees (estimated 2.9% + $0.30 per transaction)
        $paymentFees = Capsule::select("
            SELECT 
                COUNT(*) as transactions,
                SUM(total) as volume
            FROM tblinvoices
            WHERE status = 'Paid'
            AND datepaid >= ?
        ", [$startDate])[0] ?? ['transactions' => 0, 'volume' => 0];

        $estimatedFees = ($paymentFees->volume * 0.029) + ($paymentFees->transactions * 0.30);

        return [
            'domains' => round($domainCosts->annual_cost / 12, 2),
            'payment_processing' => round($estimatedFees, 2),
            'ssl_certificates' => self::getSSLCosts(),
            'total' => round(($domainCosts->annual_cost / 12) + $estimatedFees + self::getSSLCosts(), 2)
        ];
    }

    /**
     * Get SSL certificate costs
     */
    private static function getSSLCosts(): float
    {
        // Count SSL purchases
        $sslCount = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->where('dedicatedip', '1')
            ->count();

        // Assume $50/year per dedicated IP SSL
        return round($sslCount * (50 / 12), 2);
    }

    /**
     * Calculate total cost
     */
    private static function calculateTotalCost(string $period): array
    {
        $costs = [
            'compute' => self::getComputeCosts($period)['total'],
            'storage' => self::getStorageCosts($period)['total'],
            'network' => self::getNetworkCosts($period)['total'],
            'third_party' => self::getThirdPartyCosts($period)['total']
        ];

        $total = array_sum($costs);

        return [
            'compute' => $costs['compute'],
            'storage' => $costs['storage'],
            'network' => $costs['network'],
            'third_party' => $costs['third_party'],
            'total_monthly' => round($total, 2),
            'total_annual' => round($total * 12, 2)
        ];
    }

    /**
     * Get costs grouped by service type
     */
    private static function getCostsByService(): array
    {
        return Capsule::select("
            SELECT 
                p.name as product_name,
                COUNT(h.id) as services,
                SUM(h.amount) as monthly_revenue,
                SUM(h.amount) * 0.7 as estimated_cost,
                (SUM(h.amount) * 0.3) as margin
            FROM tblhosting h
            INNER JOIN tblproducts p ON h.packageid = p.id
            WHERE h.domainstatus = 'Active'
            GROUP BY p.id, p.name
            ORDER BY margin DESC
        ");
    }
}
```

### Step 3: Cost Optimization Strategies

```php
<?php
// /var/www/html/whmcs/includes/helpers/CostOptimizer.php

namespace WHMCS\Helpers;

class CostOptimizer
{
    /**
     * Analyze optimization opportunities
     */
    public static function analyzeOptimizationOpportunities(): array
    {
        return [
            'right_sizing' => self::analyzeRightSizing(),
            'reserved_capacity' => self::analyzeReservedCapacity(),
            'spot_instances' => self::analyzeSpotOpportunities(),
            'storage_tiering' => self::analyzeStorageTiering(),
            'idle_resources' => self::findIdleResources()
        ];
    }

    /**
     * Analyze right-sizing opportunities
     */
    private static function analyzeRightSizing(): array
    {
        // Find over-provisioned servers
        $servers = Capsule::select("
            SELECT 
                s.id,
                s.name,
                s.ipaddress,
                COUNT(h.id) as services,
                AVG(h.amount) as avg_service_value,
                s.monthly_cost
            FROM tblservers s
            LEFT JOIN tblhosting h ON s.id = h.server AND h.domainstatus = 'Active'
            GROUP BY s.id
            HAVING services < 5
        ");

        $opportunities = [];
        foreach ($servers as $server) {
            $utilization = ($server->services / 10) * 100; // Assume 10 is optimal
            if ($utilization < 50) {
                $opportunities[] = [
                    'server_id' => $server->id,
                    'server_name' => $server->name,
                    'current_utilization' => $utilization,
                    'monthly_savings' => $server->monthly_cost * 0.3, // 30% savings potential
                    'recommendation' => 'Consider migrating services to consolidate'
                ];
            }
        }

        return $opportunities;
    }

    /**
     * Analyze reserved capacity opportunities
     */
    private static function analyzeReservedCapacity(): array
    {
        // Calculate potential savings with reserved instances
        $computeCosts = \WHMCS\Helpers\CostTracker::getComputeCosts('30 days');

        return [
            'current_monthly_compute' => $computeCosts['total'],
            'reserved_savings' => round($computeCosts['total'] * 0.35, 2), // 35% savings
            'annual_commitment' => round($computeCosts['total'] * 12 * 0.65, 2), // 65% of annual
            'recommendation' => 'Consider 1-year reserved instances for stable workloads'
        ];
    }

    /**
     * Analyze spot/preempt instance opportunities
     */
    private static function analyzeSpotOpportunities(): array
    {
        // Non-critical workloads suitable for spot
        $batchJobs = Capsule::table('tbltask_queue')
            ->where('priority', 'low')
            ->where('status', 'pending')
            ->count();

        return [
            'suitable_workloads' => $batchJobs,
            'estimated_savings' => round($batchJobs * 5, 2), // $5 per job average
            'recommendation' => 'Run batch jobs on spot instances for 70% cost reduction'
        ];
    }

    /**
     * Analyze storage tiering
     */
    private static function analyzeStorageTiering(): array
    {
        // Find old data that can be moved to cheaper storage
        $oldData = Capsule::select("
            SELECT 
                COUNT(*) as records,
                'inactive_services' as data_type,
                SUM(COALESCE(diskusage, 0)) as size_gb
            FROM tblhosting
            WHERE domainstatus IN ('Suspended', 'Terminated')
            AND last_update < DATE_SUB(NOW(), INTERVAL 90 DAY)
        ")[0] ?? null;

        $coldStorageSavings = $oldData ? ($oldData->size_gb / 1024) * 0.02 : 0; // 80% cheaper

        return [
            'cold_data_gb' => $oldData ? round($oldData->size_gb / 1024, 2) : 0,
            'monthly_savings' => round($coldStorageSavings, 2),
            'recommendation' => 'Move inactive data to cold storage tier'
        ];
    }

    /**
     * Find idle resources
     */
    private static function findIdleResources(): array
    {
        // Find servers with no active services
        $idleServers = Capsule::select("
            SELECT s.id, s.name, s.ipaddress, s.monthly_cost
            FROM tblservers s
            LEFT JOIN tblhosting h ON s.id = h.server AND h.domainstatus = 'Active'
            GROUP BY s.id
            HAVING COUNT(h.id) = 0
        ");

        $totalIdleCost = array_sum(array_column($idleServers, 'monthly_cost'));

        return [
            'idle_server_count' => count($idleServers),
            'idle_servers' => $idleServers,
            'monthly_waste' => round($totalIdleCost, 2),
            'recommendation' => 'Decommission idle servers to reduce costs'
        ];
    }
}
```

### Step 4: Resource Monitoring

```php
<?php
// /var/www/html/whmcs/includes/hooks/cost_monitoring.php

/**
 * Daily cost monitoring
 */
add_hook('DailyCronJob', 1, function() {
    $tracker = new \WHMCS\Helpers\CostTracker();
    $optimizer = new \WHMCS\Helpers\CostOptimizer();

    // Calculate current costs
    $costs = $tracker->trackInfrastructureCost('30 days');

    // Log cost snapshot
    Capsule::table('tblcost_snapshots')->insert([
        'snapshot_date' => date('Y-m-d'),
        'total_monthly' => $costs['total']['total_monthly'],
        'compute_cost' => $costs['total']['compute'],
        'storage_cost' => $costs['total']['storage'],
        'network_cost' => $costs['total']['network'],
        'other_cost' => $costs['total']['third_party'],
        'active_services' => Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count(),
        'cost_per_service' => $costs['total']['total_monthly'] / max(1, Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count()),
        'created_at' => date('Y-m-d H:i:s')
    ]);

    // Check for optimization opportunities
    $opportunities = $optimizer->analyzeOptimizationOpportunities();

    // Alert on high costs
    $avgDailyCost = $costs['total']['total_monthly'] / 30;
    $dailyBudget = 100; // $100/day budget example

    if ($avgDailyCost > $dailyBudget) {
        logActivity("Cost Alert: Daily average cost (\${$avgDailyCost}) exceeds budget (\${$dailyBudget})");
    }

    // Log opportunities
    if (!empty($opportunities['idle_resources']['idle_servers'])) {
        $savings = $opportunities['idle_resources']['monthly_waste'];
        logActivity("Cost Optimization: Found {$opportunities['idle_resources']['idle_server_count']} idle servers costing \${$savings}/month");
    }
});

/**
 * Weekly cost report
 */
add_hook('WeeklyCronJob', 1, function() {
    $tracker = new \WHMCS\Helpers\CostTracker();

    // Generate weekly cost report
    $weeklyCosts = $tracker->trackInfrastructureCost('7 days');
    $monthlyCosts = $tracker->trackInfrastructureCost('30 days');

    $report = [
        'week_start' => date('Y-m-d', strtotime('-7 days')),
        'week_end' => date('Y-m-d'),
        'weekly_total' => $weeklyCosts['total']['total_monthly'] / 4.3,
        'monthly_projected' => $monthlyCosts['total']['total_monthly'],
        'cost_trend' => 'stable'
    ];

    // Send report to admin
    sendEmail('WeeklyCostReport', null, [
        'report' => $report,
        'recommendations' => \WHMCS\Helpers\CostOptimizer::analyzeOptimizationOpportunities()
    ]);
});
```

### Step 5: Cost Dashboard Widget

```php
<?php
// /var/www/html/whmcs/modules/widgets/CostManagementDashboard.php

namespace WHMCS\Module\Widget;

class CostManagementDashboard extends \WHMCS\Module\AbstractWidget
{
    protected $title = 'Cost Management';
    protected $author = 'System';
    protected $type = 'dashboard';

    public function getData(): array
    {
        $tracker = new \WHMCS\Helpers\CostTracker();
        $optimizer = new \WHMCS\Helpers\CostOptimizer();

        return [
            'current_costs' => $tracker->trackInfrastructureCost('30 days'),
            'opportunities' => $optimizer->analyzeOptimizationOpportunities(),
            'trend' => $this->getCostTrend(),
            'cost_per_service' => $this->calculateCostPerService()
        ];
    }

    private function getCostTrend(int $days = 30): array
    {
        $snapshots = Capsule::select("
            SELECT 
                snapshot_date,
                total_monthly,
                cost_per_service
            FROM tblcost_snapshots
            WHERE snapshot_date >= DATE_SUB(NOW(), INTERVAL ? DAY)
            ORDER BY snapshot_date
        ", [$days]);

        return $snapshots;
    }

    private function calculateCostPerService(): float
    {
        $costs = Capsule::table('tblcost_snapshots')
            ->orderBy('snapshot_date', 'desc')
            ->first();

        if (!$costs || $costs->active_services == 0) {
            return 0;
        }

        return round($costs->cost_per_service, 2);
    }

    public function generateOutput($data): string
    {
        $costs = $data['current_costs'];
        $totalMonthly = $costs['total']['total_monthly'];
        $computeSavings = $data['opportunities']['reserved_capacity']['reserved_savings'] ?? 0;
        $idleWaste = $data['opportunities']['idle_resources']['monthly_waste'] ?? 0;
        $totalSavings = $computeSavings + $idleWaste;

        return <<<HTML
<div class="cost-dashboard">
    <div class="cost-summary">
        <div class="cost-card total">
            <span class="value">\${$this->formatNumber($totalMonthly)}</span>
            <span class="label">Monthly Cost</span>
        </div>
        <div class="cost-card compute">
            <span class="value">\${$this->formatNumber($costs['total']['compute'])}</span>
            <span class="label">Compute</span>
        </div>
        <div class="cost-card storage">
            <span class="value">\${$this->formatNumber($costs['total']['storage'])}</span>
            <span class="label">Storage</span>
        </div>
    </div>

    <div class="savings-section">
        <h4>Optimization Opportunities</h4>
        <div class="savings-row">
            <span class="savings-label">Potential Monthly Savings:</span>
            <span class="savings-value">\${$this->formatNumber($totalSavings)}</span>
        </div>
        <ul class="opportunities-list">
            <li>Reserved capacity: \${$this->formatNumber($computeSavings)}</li>
            <li>Idle resources: \${$this->formatNumber($idleWaste)}</li>
        </ul>
    </div>

    <div class="cost-per-service">
        <span class="label">Cost per Service:</span>
        <span class="value">\${$data['cost_per_service']}</span>
    </div>
</div>
HTML;
    }

    private function formatNumber(float $num): string
    {
        return number_format($num, 2);
    }
}
```

### Step 6: Cost Alert Configuration

```php
<?php
// /var/www/html/whmcs/includes/helpers/CostAlerts.php

namespace WHMCS\Helpers;

class CostAlerts
{
    private $thresholds = [
        'daily_budget' => 100,      // $100/day
        'weekly_budget' => 650,     // $650/week
        'monthly_budget' => 2500,    // $2500/month
        'per_service_max' => 50,     // $50/service max
        'growth_threshold' => 20     // 20% increase threshold
    ];

    /**
     * Check cost thresholds
     */
    public static function checkThresholds(): array
    {
        $tracker = new CostTracker();
        $costs = $tracker->trackInfrastructureCost('30 days');

        $alerts = [];

        // Check monthly budget
        if ($costs['total']['total_monthly'] > self::getThreshold('monthly_budget')) {
            $alerts[] = [
                'level' => 'critical',
                'message' => 'Monthly cost exceeds budget by ' . 
                    round($costs['total']['total_monthly'] - self::getThreshold('monthly_budget'), 2)
            ];
        }

        // Check per-service cost
        $costPerService = $costs['total']['total_monthly'] / max(1, 
            Capsule::table('tblhosting')->where('domainstatus', 'Active')->count());

        if ($costPerService > self::getThreshold('per_service_max')) {
            $alerts[] = [
                'level' => 'warning',
                'message' => 'Cost per service (\$' . round($costPerService, 2) . ') exceeds threshold'
            ];
        }

        // Check growth rate
        $trend = self::getCostTrend();
        if (count($trend) >= 7) {
            $recentAvg = array_sum(array_slice(array_column($trend, 'total_monthly'), -7)) / 7;
            $olderAvg = array_sum(array_slice(array_column($trend, 'total_monthly'), -14, 7)) / 7;

            if ($olderAvg > 0) {
                $growth = (($recentAvg - $olderAvg) / $olderAvg) * 100;

                if ($growth > self::getThreshold('growth_threshold')) {
                    $alerts[] = [
                        'level' => 'warning',
                        'message' => 'Cost growth rate (' . round($growth, 1) . '%) exceeds threshold'
                    ];
                }
            }
        }

        return $alerts;
    }

    private static function getThreshold(string $key): float
    {
        return (new self())->thresholds[$key] ?? 0;
    }

    private static function getCostTrend(): array
    {
        return Capsule::select("
            SELECT total_monthly
            FROM tblcost_snapshots
            ORDER BY snapshot_date DESC
            LIMIT 30
        ");
    }

    /**
     * Send cost alerts
     */
    public static function sendAlerts(): void
    {
        $alerts = self::checkThresholds();

        foreach ($alerts as $alert) {
            if ($alert['level'] === 'critical') {
                // Immediate alert
                sendEmail('CostCriticalAlert', null, [
                    'alert_message' => $alert['message']
                ]);
            } else {
                // Warning - include in daily report
                logActivity("Cost Warning: {$alert['message']}");
            }
        }
    }
}
```

### Step 7: Cost Storage Schema

```sql
-- Create cost snapshots table
CREATE TABLE IF NOT EXISTS `tblcost_snapshots` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `snapshot_date` DATE NOT NULL,
    `total_monthly` DECIMAL(10,2) DEFAULT 0,
    `compute_cost` DECIMAL(10,2) DEFAULT 0,
    `storage_cost` DECIMAL(10,2) DEFAULT 0,
    `network_cost` DECIMAL(10,2) DEFAULT 0,
    `other_cost` DECIMAL(10,2) DEFAULT 0,
    `active_services` INT DEFAULT 0,
    `cost_per_service` DECIMAL(10,2) DEFAULT 0,
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY `idx_snapshot_date` (`snapshot_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Create cost budgets table
CREATE TABLE IF NOT EXISTS `tblcost_budgets` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `budget_name` VARCHAR(100) NOT NULL,
    `period` ENUM('daily', 'weekly', 'monthly') DEFAULT 'monthly',
    `amount` DECIMAL(10,2) NOT NULL,
    `alert_threshold` DECIMAL(5,2) DEFAULT 80,
    `is_active` TINYINT(1) DEFAULT 1,
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Create optimization recommendations table
CREATE TABLE IF NOT EXISTS `tblcost_recommendations` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `recommendation_type` VARCHAR(50) NOT NULL,
    `description` TEXT,
    `potential_savings` DECIMAL(10,2) DEFAULT 0,
    `status` ENUM('pending', 'approved', 'implemented', 'dismissed') DEFAULT 'pending',
    `impact_score` INT DEFAULT 0,
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Cost Optimization Summary

| Category | Monthly Cost | Savings Potential | Priority |
|----------|--------------|-------------------|----------|
| Compute | Variable | 30-40% | High |
| Storage | $XX | 50-70% | Medium |
| Network | $XX | 20-30% | Medium |
| Reserved Instances | N/A | 35% | High |
| Idle Resources | $XX | 100% | Critical |

## Best Practices

1. **Right-Sizing**: Match server specs to actual workload needs
2. **Use Reserved Capacity**: Commit for stable workloads to save 30-60%
3. **Auto-Scaling**: Scale resources based on demand
4. **Spot/Preempt**: Use for non-critical, interruptible workloads
5. **Storage Tiering**: Move cold data to cheaper storage
6. **Monitor Continuously**: Track costs daily for anomalies
7. **Set Budgets**: Establish cost budgets with alerts

## Common Pitfalls

- **Over-Provisioning**: Buying more resources than needed
- **No Monitoring**: Not tracking costs until surprise bills
- **Static Allocation**: Not adjusting resources with demand
- **Ignoring Idle Resources**: Paying for unused servers
- **Not Using Reserved**: Paying on-demand rates unnecessarily
- **No Cost Attribution**: Not knowing which services cost what

## Verification Checklist

- [ ] Cost tracking implemented
- [ ] Optimization analysis configured
- [ ] Cost dashboard widget created
- [ ] Alert thresholds set
- [ ] Weekly reports scheduled
- [ ] Budget targets established
- [ ] Savings tracked over time

## Related Documentation

- [WHMCS Cost Optimization](whmcs-cost-optimization.md)
- [WHMCS Metrics Visualization](whmcs-metrics-visualization.md)
- [WHMCS Scaling Guide](whmcs-scaling-guide.md)