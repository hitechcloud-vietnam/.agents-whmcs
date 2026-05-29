# WHMCS Capacity Planning Workflow

## Overview
This workflow helps with capacity planning and resource allocation for WHMCS.

## Step 1: Capacity Planning Service

```php
<?php
// src/Service/CapacityPlanningService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class CapacityPlanningService
{
    public function generateCapacityReport(): array
    {
        return [
            'generated_at' => date('Y-m-d H:i:s'),
            'current_usage' => $this->getCurrentUsage(),
            'projections' => $this->getProjections(),
            'recommendations' => $this->getRecommendations()
        ];
    }

    private function getCurrentUsage(): array
    {
        // Database size
        $dbSize = Capsule::connection()->select(
            "SELECT ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size FROM information_schema.tables WHERE table_schema = ?",
            [Capsule::config('db_name')]
        );

        // Client counts
        $totalClients = Capsule::table('tblclients')->count();
        $activeClients = Capsule::table('tblclients')->where('status', 'Active')->count();

        // Service counts
        $totalServices = Capsule::table('tblhosting')->count();
        $activeServices = Capsule::table('tblhosting')->where('domainstatus', 'Active')->count();

        // Disk usage
        $diskUsed = disk_total_space('/') - disk_free_space('/');
        $diskTotal = disk_total_space('/');

        return [
            'database_size_mb' => $dbSize[0]->size ?? 0,
            'total_clients' => $totalClients,
            'active_clients' => $activeClients,
            'total_services' => $totalServices,
            'active_services' => $activeServices,
            'disk_used_gb' => round($diskUsed / 1024 / 1024 / 1024, 2),
            'disk_total_gb' => round($diskTotal / 1024 / 1024 / 1024, 2),
            'disk_percent_used' => round(($diskUsed / $diskTotal) * 100, 2)
        ];
    }

    private function getProjections(): array
    {
        $currentUsage = $this->getCurrentUsage();

        // Get growth rates from last 30 days
        $lastMonthClients = Capsule::table('tblclients')
            ->where('datecreated', '>=', date('Y-m-d', strtotime('-30 days')))
            ->count();

        $lastMonthServices = Capsule::table('tblhosting')
            ->where('regdate', '>=', date('Y-m-d', strtotime('-30 days')))
            ->count();

        // Daily growth rates
        $dailyClientGrowth = $lastMonthClients / 30;
        $dailyServiceGrowth = $lastMonthServices / 30;

        // 90-day projections
        $projectedClients = $currentUsage['active_clients'] + ($dailyClientGrowth * 90);
        $projectedServices = $currentUsage['active_services'] + ($dailyServiceGrowth * 90);

        // Database growth projection (estimate 10MB per 100 new clients)
        $dbGrowthPerClient = 0.1; // MB
        $projectedDbGrowth = $projectedClients * $dbGrowthPerClient;

        return [
            'daily_client_growth' => round($dailyClientGrowth, 2),
            'daily_service_growth' => round($dailyServiceGrowth, 2),
            'projected_clients_90d' => round($projectedClients),
            'projected_services_90d' => round($projectedServices),
            'projected_db_growth_mb' => round($projectedDbGrowth, 2),
            'disk_usage_90d_gb' => round(
                ($currentUsage['disk_used_gb'] * 90 * $dailyServiceGrowth * 0.5) / 1024,
                2
            )
        ];
    }

    private function getRecommendations(): array
    {
        $currentUsage = $this->getCurrentUsage();
        $recommendations = [];

        // Disk space
        if ($currentUsage['disk_percent_used'] > 80) {
            $recommendations[] = [
                'priority' => 'high',
                'category' => 'disk',
                'issue' => 'Disk space usage above 80%',
                'recommendation' => 'Consider adding disk space or archiving old data'
            ];
        }

        // Database size
        if ($currentUsage['database_size_mb'] > 5000) {
            $recommendations[] = [
                'priority' => 'medium',
                'category' => 'database',
                'issue' => 'Database size exceeds 5GB',
                'recommendation' => 'Consider database optimization and archiving old records'
            ];
        }

        // Client scaling
        if ($currentUsage['total_clients'] > 5000) {
            $recommendations[] = [
                'priority' => 'medium',
                'category' => 'scaling',
                'issue' => 'Client count above 5000',
                'recommendation' => 'Consider implementing read replicas and caching'
            ];
        }

        return $recommendations;
    }

    public function planResourceUpgrade(array $targetMetrics): array
    {
        $currentUsage = $this->getCurrentUsage();

        return [
            'current_resources' => $currentUsage,
            'target_metrics' => $targetMetrics,
            'required_upgrades' => $this->calculateRequiredUpgrades($currentUsage, $targetMetrics),
            'estimated_cost' => $this->estimateUpgradeCost($targetMetrics)
        ];
    }

    private function calculateRequiredUpgrades(array $current, array $target): array
    {
        $upgrades = [];

        if (isset($target['min_disk_gb']) && $current['disk_total_gb'] < $target['min_disk_gb']) {
            $upgrades[] = [
                'type' => 'disk',
                'current' => $current['disk_total_gb'] . ' GB',
                'recommended' => $target['min_disk_gb'] . ' GB',
                'action' => 'Add ' . ($target['min_disk_gb'] - $current['disk_total_gb']) . ' GB disk space'
            ];
        }

        if (isset($target['min_db_size_mb']) && $current['database_size_mb'] > $target['min_db_size_mb'] * 0.8) {
            $upgrades[] = [
                'type' => 'database',
                'action' => 'Consider upgrading database server or implementing partitioning'
            ];
        }

        return $upgrades;
    }

    private function estimateUpgradeCost(array $target): array
    {
        $costs = [];

        if (isset($target['min_disk_gb'])) {
            $costs['disk_100gb_monthly'] = 10; // Estimated monthly cost per 100GB
        }

        if (isset($target['min_clients'])) {
            $costs['additional_instances'] = max(0, ceil(($target['min_clients'] - 5000) / 5000));
        }

        return $costs;
    }
}
```

## Verification Checklist

- [ ] Capacity planning service implemented
- [ ] Current usage tracking working
- [ ] Growth projections calculating
- [ ] Recommendations generating
- [ ] Resource upgrade planning working
