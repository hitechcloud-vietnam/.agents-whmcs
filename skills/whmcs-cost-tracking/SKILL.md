# WHMCS Cost Tracking Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for cost tracking, margin calculation, and analytics for WHMCS services.

## When to Use

- Building cost analysis dashboards
- Tracking service margins
- Analyzing profitability
- Setting up billing cost reports

## Cost Tracking Patterns

### 1. Cost Calculation Engine

```php
<?php
namespace WHMCS\Cost;

class CostCalculator {
    public function calculateServiceCost(int $serviceId): array {
        $service = Capsule::table('tblhosting')->find($serviceId);
        if (!$service) {
            return ['error' => 'Service not found'];
        }

        $product = Capsule::table('tblproducts')->find($service->packageid);

        // Direct costs
        $directCosts = $this->getDirectCosts($service);

        // Allocation costs (shared infrastructure)
        $allocatedCosts = $this->getAllocatedCosts($service, $product);

        //蠢otal costs
        $totalCost = $directCosts['total'] + $allocatedCosts['total'];

        // Revenue and margin
        $revenue = $service->amount ?? $product->recurringmonthly;
        $margin = $revenue > 0 ? (($revenue - $totalCost) / $revenue) * 100 : 0;

        return [
            'service_id' => $serviceId,
            'revenue' => $revenue,
            'direct_costs' => $directCosts,
            'allocated_costs' => $allocatedCosts,
            'total_cost' => $totalCost,
            'margin' => round($margin, 2),
            'profit' => $revenue - $totalCost,
        ];
    }

    private function getDirectCosts(array $service): array {
        $costs = [
            'server_cost' => $this->getServerCostShare($service['server']),
            'license_cost' => $this->getLicenseCost($service['packageid']),
            'bandwidth_cost' => $this->calculateBandwidthCost($service),
            'backup_cost' => $this->calculateBackupCost($service),
        ];

        return [
            'server_cost' => $costs['server_cost'],
            'license_cost' => $costs['license_cost'],
            'bandwidth_cost' => $costs['bandwidth_cost'],
            'backup_cost' => $costs['backup_cost'],
            'total' => array_sum($costs),
        ];
    }

    private function getServerCostShare(?int $serverId): float {
        if (!$serverId) return 0;

        $server = Capsule::table('tblservers')->find($serverId);
        $monthlyBase = $server->monthlycost ?? 0;

        if ($monthlyBase == 0) return 0;

        // Count active services on this server
        $activeServices = Capsule::table('tblhosting')
            ->where('server', $serverId)
            ->where('domainstatus', 'Active')
            ->count();

        $serverRam = $server->maxmemory ?? 16384; // Default 16GB

        // Estimate resource share based on memory allocation
        $resourceShare = 0.2; // Default 20% for typical shared VM
        $serviceMemory = 2048; // Default 2GB per service

        $estimatedShare = $serviceMemory / $serverRam;
        $costShare = ($monthlyBase / $activeServices) * (1 + $estimatedShare);

        return round($costShare, 4);
    }

    private function getLicenseCost(int $productId): float {
        $license = Capsule::table('mod_license_costs')
            ->where('product_id', $productId)
            ->first();

        return $license ? (float) $license->monthly_cost : 0;
    }

    private function calculateBandwidthCost(array $service): float {
        $usage = Capsule::table('mod_bandwidth_usage')
            ->where('service_id', $service['id'])
            ->where('month', date('Y-m'))
            ->sum('bytes_used');

        $gbUsed = $usage / (1024 * 1024 * 1024);
        $costPerGb = $this->getBandwidthCostPerGb();

        return round($gbUsed * $costPerGb, 4);
    }

    private function calculateBackupCost(array $service): float {
        $backupSize = $service['storage'] ?? 0; // GB
        $costPerGb = $this->getBackupCostPerGb();

        return round($backupSize * $costPerGb, 4);
    }

    private function getAllocatedCosts(array $service, array $product): array {
        $costs = [
            'support_cost' => $this->calculateSupportCost($product),
            'management_cost' => $this->calculateManagementCost($service),
            'overhead_cost' => $this->calculateOverheadCost($service),
        ];

        return $costs + ['total' => array_sum($costs)];
    }

    private function calculateSupportCost(array $product): float {
        $tier = $product['supporttype'] ?? 1;

        return match($tier) {
            1 => 2.50,  // Basic
            2 => 5.00,  // Standard
            3 => 10.00, // Premium
            default => 2.50,
        };
    }

    private function calculateManagementCost(array $service): float {
        // Management overhead per service type
        $managementRates = [
            'reseller' => 5.00,
            'vps' => 3.00,
            'dedicated' => 10.00,
            'shared' => 1.50,
        ];

        $serviceType = $service['servertype'] ?? 'shared';
        return $managementRates[$serviceType] ?? 1.50;
    }

    private function calculateOverheadCost(array $service): float {
        // Platform overhead (billing system, control panel, etc.)
        return 1.00;
    }

    private function getBandwidthCostPerGb(): float {
        $setting = Capsule::table('tblconfiguration')
            ->where('setting', 'bandwidth_cost_per_gb')
            ->first();

        return $setting ? (float) $setting->value : 0.05;
    }

    private function getBackupCostPerGb(): float {
        $setting = Capsule::table('tblconfiguration')
            ->where('setting', 'backup_cost_per_gb')
            ->first();

        return $setting ? (float) $setting->value : 0.10;
    }
}
```

### 2. Cost Analytics Dashboard

```php
<?php
namespace WHMCS\Cost;

class CostAnalytics {
    public function getProfitabilityReport(array $dateRange): array {
        $services = Capsule::table('tblhosting')
            ->selectRaw('tblhosting.*, tblproducts.name as product_name, tblproductgroups.name as group_name')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->join('tblproductgroups', 'tblproducts.gid', '=', 'tblproductgroups.id')
            ->where('tblhosting.domainstatus', 'Active')
            ->get();

        $calculator = new CostCalculator();
        $report = [
            'summary' => [
                'total_services' => 0,
                'total_revenue' => 0,
                'total_cost' => 0,
                'total_profit' => 0,
                'average_margin' => 0,
            ],
            'by_group' => [],
            'by_product' => [],
        ];

        $totalMargin = 0;
        foreach ($services as $service) {
            $costData = $calculator->calculateServiceCost($service['id']);

            $report['summary']['total_services']++;
            $report['summary']['total_revenue'] += $costData['revenue'];
            $report['summary']['total_cost'] += $costData['total_cost'];
            $report['summary']['total_profit'] += $costData['profit'];
            $totalMargin += $costData['margin'];

            // Group by product group
            $groupName = $costData['group_name'] ?? 'Ungrouped';
            if (!isset($report['by_group'][$groupName])) {
                $report['by_group'][$groupName] = [
                    'services' => 0,
                    'revenue' => 0,
                    'cost' => 0,
                    'profit' => 0,
                    'margin' => 0,
                ];
            }
            $report['by_group'][$groupName]['services']++;
            $report['by_group'][$groupName]['revenue'] += $costData['revenue'];
            $report['by_group'][$groupName]['cost'] += $costData['total_cost'];
            $report['by_group'][$groupName]['profit'] += $costData['profit'];
        }

        // Calculate margins per group
        foreach ($report['by_group'] as &$group) {
            $group['margin'] = $group['revenue'] > 0
                ? round(($group['profit'] / $group['revenue']) * 100, 2)
                : 0;
        }

        $report['summary']['average_margin'] = $report['summary']['total_services'] > 0
            ? round($totalMargin / $report['summary']['total_services'], 2)
            : 0;

        return $report;
    }

    public function getTrendReport(int $months = 12): array {
        $trends = [];
        $calculator = new CostCalculator();

        for ($i = 0; $i < $months; $i++) {
            $month = date('Y-m', strtotime("-$i months"));
            $services = Capsule::table('tblhosting')
                ->where('domainstatus', 'Active')
                ->get();

            $totalRevenue = 0;
            $totalCost = 0;

            foreach ($services as $service) {
                $costData = $calculator->calculateServiceCost($service['id']);
                $totalRevenue += $costData['revenue'];
                $totalCost += $costData['total_cost'];
            }

            $trends[] = [
                'month' => $month,
                'services' => count($services),
                'revenue' => $totalRevenue,
                'cost' => $totalCost,
                'profit' => $totalRevenue - $totalCost,
                'margin' => $totalRevenue > 0
                    ? round((($totalRevenue - $totalCost) / $totalRevenue) * 100, 2)
                    : 0,
            ];
        }

        return array_reverse($trends);
    }

    public function findLowMarginServices(float $threshold = 20): array {
        $services = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->get();

        $calculator = new CostCalculator();
        $lowMargin = [];

        foreach ($services as $service) {
            $costData = $calculator->calculateServiceCost($service['id']);

            if ($costData['margin'] < $threshold && $costData['margin'] >= 0) {
                $lowMargin[] = [
                    'service_id' => $service['id'],
                    'domain' => $service['domain'],
                    'revenue' => $costData['revenue'],
                    'cost' => $costData['total_cost'],
                    'margin' => $costData['margin'],
                    'client' => $this->getClientInfo($service['userid']),
                ];
            }
        }

        usort($lowMargin, fn($a, $b) => $a['margin'] <=> $b['margin']);

        return $lowMargin;
    }

    private function getClientInfo(int $userId): array {
        $client = Capsule::table('tblclients')->find($userId);
        return [
            'id' => $client['id'],
            'name' => trim($client['firstname'] . ' ' . $client['lastname']),
            'email' => $client['email'],
        ];
    }
}
```

### 3. Cost Allocation Tables

```php
<?php
// Database schema for cost tracking
function createCostTrackingTables(): void {
    Capsule::schema()->create('mod_server_costs', function($t) {
        $t->increments('id');
        $t->integer('server_id')->unsigned();
        $t->decimal('monthly_base_cost', 10, 4)->default(0);
        $t->decimal('monthly_license_cost', 10, 4)->default(0);
        $t->integer('max_memory')->unsigned()->default(16384);
        $t->integer('max_disk')->unsigned()->default(0);
        $t->integer('max_bandwidth')->unsigned()->default(0);
        $t->timestamps();

        $t->foreign('server_id')->references('id')->on('tblservers')->onDelete('cascade');
    });

    Capsule::schema()->create('mod_license_costs', function($t) {
        $t->increments('id');
        $t->integer('product_id')->unsigned();
        $t->string('license_type'); // cpanel, plesk, etc.
        $t->decimal('monthly_cost', 10, 4)->default(0);
        $t->timestamps();

        $t->foreign('product_id')->references('id')->on('tblproducts')->onDelete('cascade');
    });

    Capsule::schema()->create('mod_bandwidth_usage', function($t) {
        $t->increments('id');
        $t->integer('service_id')->unsigned();
        $t->string('month', 7); // YYYY-MM
        $t->bigInteger('bytes_used')->unsigned()->default(0);
        $t->timestamps();

        $t->foreign('service_id')->references('id')->on('tblhosting')->onDelete('cascade');
        $t->unique(['service_id', 'month']);
    });

    Capsule::schema()->create('mod_cost_reports', function($t) {
        $t->increments('id');
        $t->string('month', 7);
        $t->decimal('total_revenue', 12, 4)->default(0);
        $t->decimal('total_cost', 12, 4)->default(0);
        $t->decimal('total_profit', 12, 4)->default(0);
        $t->decimal('average_margin', 5, 2)->default(0);
        $t->json('breakdown');
        $t->timestamps();

        $t->unique(['month']);
    });
}
```

### 4. Pricing Optimization

```php
<?php
namespace WHMCS\Cost;

class PricingOptimizer {
    public function suggestPriceAdjustment(int $serviceId): array {
        $calculator = new CostCalculator();
        $costData = $calculator->calculateServiceCost($serviceId);

        $targetMargin = $this->getTargetMargin();
        $currentMargin = $costData['margin'];

        if ($currentMargin < $targetMargin) {
            $suggestedPrice = $this->calculatePriceForMargin($costData['total_cost'], $targetMargin);
            $priceIncrease = $suggestedPrice - $costData['revenue'];
            $percentIncrease = ($priceIncrease / $costData['revenue']) * 100;

            return [
                'action' => 'increase',
                'current_price' => $costData['revenue'],
                'suggested_price' => round($suggestedPrice, 2),
                'price_increase' => round($priceIncrease, 2),
                'percent_increase' => round($percentIncrease, 1),
                'reason' => "Margin {$currentMargin}% is below target {$targetMargin}%",
            ];
        }

        return [
            'action' => 'maintain',
            'current_price' => $costData['revenue'],
            'current_margin' => $currentMargin,
            'target_margin' => $targetMargin,
            'reason' => 'Margin is within acceptable range',
        ];
    }

    public function bulkPriceAnalysis(): array {
        $services = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->get();

        $calculator = new CostCalculator();
        $analysis = [
            'optimize_up' => [],
            'maintain' => [],
            'review_needed' => [],
        ];

        foreach ($services as $service) {
            $costData = $calculator->calculateServiceCost($service['id']);
            $suggestion = $this->suggestPriceAdjustment($service['id']);

            $serviceData = [
                'service_id' => $service['id'],
                'domain' => $service['domain'],
                'revenue' => $costData['revenue'],
                'cost' => $costData['total_cost'],
                'margin' => $costData['margin'],
            ] + $suggestion;

            if ($suggestion['action'] === 'increase') {
                if ($suggestion['percent_increase'] > 20) {
                    $analysis['review_needed'][] = $serviceData;
                } else {
                    $analysis['optimize_up'][] = $serviceData;
                }
            } else {
                $analysis['maintain'][] = $serviceData;
            }
        }

        return $analysis;
    }

    private function getTargetMargin(): float {
        $setting = Capsule::table('tblconfiguration')
            ->where('setting', 'target_margin_percent')
            ->first();

        return $setting ? (float) $setting->value : 30;
    }

    private function calculatePriceForMargin(float $cost, float $targetMargin): float {
        // price = cost / (1 - margin_percentage)
        return $cost / (1 - ($targetMargin / 100));
    }
}
```

## Hook Integration

```php
// Track costs on service changes
add_hook('AfterModuleCreate', 1, function($vars) {
    $calculator = new CostCalculator();
    $costData = $calculator->calculateServiceCost($vars['serviceid']);

    logModuleChange('cost', [
        'service_id' => $vars['serviceid'],
        'revenue' => $costData['revenue'],
        'cost' => $costData['total_cost'],
        'margin' => $costData['margin'],
    ]);
});

// Flag low margin services
add_hook('DailyCronJob', 1, function($vars) {
    $analytics = new CostAnalytics();
    $lowMargin = $analytics->findLowMarginServices(15);

    if (count($lowMargin) > 0) {
        sendAdminNotification('warning', 'Low margin services detected', [
            'count' => count($lowMargin),
            'services' => array_slice($lowMargin, 0, 10),
        ]);
    }
});
```

## Checklist

- [ ] Cost tracking database schema
- [ ] Direct cost calculation (server, license, bandwidth, backup)
- [ ] Allocated cost calculation (support, management, overhead)
- [ ] Profitability reporting
- [ ] Trend analysis
- [ ] Low margin detection
- [ ] Price optimization suggestions
- [ ] Admin notifications

---

**Related Skills:**
- whmcs-service-billing
- whmcs-reporting
- whmcs-payment-analytics
- whmcs-metrics-analytics
