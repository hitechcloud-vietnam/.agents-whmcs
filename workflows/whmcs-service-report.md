# WHMCS Service Report Workflow

## Overview
This workflow generates service usage and performance reports.

## Step-by-Step Process

```php
<?php
// /includes/reports/ServiceReportGenerator.php

class ServiceReportGenerator
{
    public function generate(array $params = []): array
    {
        return [
            'summary' => $this->getServiceSummary(),
            'by_product' => $this->getByProduct(),
            'usage_stats' => $this->getUsageStats(),
            'renewal_rates' => $this->getRenewalRates()
        ];
    }

    private function getServiceSummary(): array
    {
        return [
            'total_services' => Capsule::table('tblhosting')->count(),
            'active' => Capsule::table('tblhosting')->where('domainstatus', 'Active')->count(),
            'suspended' => Capsule::table('tblhosting')->where('domainstatus', 'Suspended')->count(),
            'terminated' => Capsule::table('tblhosting')->where('domainstatus', 'Terminated')->count()
        ];
    }

    private function getByProduct(): array
    {
        return Capsule::select("
            SELECT
                p.name as product_name,
                p.type,
                COUNT(h.id) as total,
                SUM(CASE WHEN h.domainstatus = 'Active' THEN 1 ELSE 0 END) as active,
                SUM(h.amount) as mrr
            FROM tblproducts p
            LEFT JOIN tblhosting h ON p.id = h.packageid
            GROUP BY p.id
            ORDER BY total DESC
        ");
    }

    private function getUsageStats(): array
    {
        return Capsule::select("
            SELECT
                h.domain,
                h.diskusage,
                h.bandwidthusage,
                h.bandwidthlimit,
                ROUND((h.bandwidthusage / NULLIF(h.bandwidthlimit, 0)) * 100, 1) as bandwidth_percent
            FROM tblhosting h
            WHERE h.domainstatus = 'Active'
            AND h.bandwidthlimit > 0
            ORDER BY bandwidth_percent DESC
            LIMIT 20
        ");
    }

    private function getRenewalRates(): array
    {
        $lastMonth = date('Y-m-d', strtotime('-30 days'));
        $lastYear = date('Y-m-d', strtotime('-365 days'));

        return Capsule::select("
            SELECT
                p.name,
                COUNT(CASE WHEN h.nextduedate >= ? AND h.nextduedate <= ? THEN 1 END) as due_soon,
                COUNT(CASE WHEN h.domainstatus = 'Active' THEN 1 END) as total,
                COUNT(CASE WHEN h.domainstatus = 'Terminated' THEN 1 END) as churned
            FROM tblproducts p
            LEFT JOIN tblhosting h ON p.id = h.packageid
            GROUP BY p.id
        ", [$lastMonth, date('Y-m-d')]);
    }
}
```

## Related Workflows
- [WHMCS Churn Report](./whmcs-churn-report.md)
- [WHMCS Revenue Report](./whmcs-revenue-report.md)