# WHMCS Cost Optimization Workflow

## Overview
This workflow helps identify and implement cost optimizations for WHMCS.

## Step 1: Cost Optimization Service

```php
<?php
// src/Service/CostOptimizationService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class CostOptimizationService
{
    public function analyzeCosts(): array
    {
        return [
            'generated_at' => date('Y-m-d H:i:s'),
            'infrastructure_costs' => $this->analyzeInfrastructureCosts(),
            'service_costs' => $this->analyzeServiceCosts(),
            'opportunities' => $this->identifyOpportunities()
        ];
    }

    private function analyzeInfrastructureCosts(): array
    {
        // Server costs (configured values)
        $serverCosts = [
            'web_server' => 50, // Monthly cost
            'database_server' => 75,
            'storage' => 20,
            'backup' => 15,
            'cdn' => 25
        ];

        // Calculate actual resource usage costs
        $dbSize = $this->getDatabaseSize();
        $storageUsed = disk_total_space('/') - disk_free_space('/');

        // Adjust storage cost based on usage
        $storageGB = round($storageUsed / 1024 / 1024 / 1024, 2);
        $actualStorageCost = min($serverCosts['storage'], $storageGB * 0.1);

        return [
            'monthly_total' => array_sum($serverCosts),
            'breakdown' => $serverCosts,
            'actual_storage_gb' => $storageGB,
            'actual_storage_cost' => $actualStorageCost
        ];
    }

    private function analyzeServiceCosts(): array
    {
        // Get unpaid invoices (potential revenue loss)
        $unpaidInvoices = Capsule::table('tblinvoices')
            ->where('status', 'Unpaid')
            ->where('duedate', '<', date('Y-m-d'))
            ->sum('total');

        // Get overdue invoices
        $overdueInvoices = Capsule::table('tblinvoices')
            ->where('status', 'Unpaid')
            ->where('duedate', '<', date('Y-m-d', strtotime('-30 days')))
            ->sum('total');

        // Service utilization
        $activeServices = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count();

        $totalServices = Capsule::table('tblhosting')
            ->count();

        $utilizationRate = $totalServices > 0
            ? round(($activeServices / $totalServices) * 100, 2)
            : 100;

        return [
            'unpaid_amount' => round($unpaidInvoices, 2),
            'overdue_amount' => round($overdueInvoices, 2),
            'service_utilization' => $utilizationRate,
            'active_services' => $activeServices,
            'total_services' => $totalServices
        ];
    }

    private function identifyOpportunities(): array
    {
        $opportunities = [];

        // Check for idle services
        $idleServices = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->where('regdate', '<', date('Y-m-d', strtotime('-90 days')))
            ->count();

        if ($idleServices > 10) {
            $opportunities[] = [
                'type' => 'service_optimization',
                'title' => 'Idle Services',
                'description' => "$idleServices services may be inactive",
                'potential_savings' => '$' . ($idleServices * 5),
                'action' => 'Review and consolidate idle services'
            ];
        }

        // Check for large attachments
        $largeAttachments = Capsule::table('tblticketattachments')
            ->selectRaw('SUM(filesize) as total_size')
            ->first();

        $totalAttachmentSize = $largeAttachments->total_size ?? 0;
        $attachmentGB = round($totalAttachmentSize / 1024 / 1024 / 1024, 2);

        if ($attachmentGB > 50) {
            $opportunities[] = [
                'type' => 'storage_optimization',
                'title' => 'Large Attachment Storage',
                'description' => "Total attachments: {$attachmentGB}GB",
                'potential_savings' => '$' . round($attachmentGB * 0.5),
                'action' => 'Consider archiving or compressing attachments'
            ];
        }

        // Check for duplicate data
        $opportunities[] = [
            'type' => 'data_optimization',
            'title' => 'Review Database Indexes',
            'description' => 'Ensure proper indexing for query optimization',
            'potential_savings' => 'Improved performance',
            'action' => 'Run database optimization'
        ];

        return $opportunities;
    }

    private function getDatabaseSize(): float
    {
        $result = Capsule::connection()->select(
            "SELECT ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size FROM information_schema.tables WHERE table_schema = ?",
            [Capsule::config('db_name')]
        );

        return $result[0]->size ?? 0;
    }

    public function getOptimizationRecommendations(): array
    {
        return [
            [
                'priority' => 'high',
                'category' => 'billing',
                'recommendation' => 'Automate invoice collection to reduce unpaid amounts',
                'estimated_impact' => '15-20% reduction in unpaid invoices'
            ],
            [
                'priority' => 'medium',
                'category' => 'storage',
                'recommendation' => 'Implement tiered storage for old data',
                'estimated_impact' => '30-40% reduction in storage costs'
            ],
            [
                'priority' => 'medium',
                'category' => 'compute',
                'recommendation' => 'Scale down during off-peak hours',
                'estimated_impact' => '20-30% reduction in compute costs'
            ],
            [
                'priority' => 'low',
                'category' => 'networking',
                'recommendation' => 'Optimize CDN usage',
                'estimated_impact' => '10-15% reduction in bandwidth costs'
            ]
        ];
    }
}
```

## Verification Checklist

- [ ] Cost analysis working
- [ ] Infrastructure costs tracking
- [ ] Service costs analyzing
- [ ] Opportunities identified
- [ ] Recommendations generating
