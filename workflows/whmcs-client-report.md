# WHMCS Client Report Workflow

## Overview
This workflow generates client activity and engagement reports.

## Step-by-Step Process

```php
<?php
// /includes/reports/ClientReportGenerator.php

class ClientReportGenerator
{
    public function generate(array $params = []): array
    {
        return [
            'summary' => $this->getClientSummary(),
            'new_clients' => $this->getNewClients(),
            'client_activity' => $this->getActivityMetrics(),
            'cohort_analysis' => $this->getCohortData(),
            'engagement' => $this->getEngagementMetrics()
        ];
    }

    private function getClientSummary(): array
    {
        return [
            'total_clients' => Capsule::table('tblclients')->count(),
            'active_clients' => Capsule::table('tblclients')->where('status', 'Active')->count(),
            'inactive_clients' => Capsule::table('tblclients')->where('status', 'Inactive')->count(),
            'with_services' => Capsule::table('tblhosting')
                ->distinct('userid')->count('userid')
        ];
    }

    private function getNewClients(): array
    {
        return Capsule::select("
            SELECT
                DATE(datecreated) as date,
                COUNT(*) as new_clients
            FROM tblclients
            WHERE datecreated >= DATE_SUB(NOW(), INTERVAL 30 DAY)
            GROUP BY DATE(datecreated)
            ORDER BY date DESC
        ");
    }

    private function getActivityMetrics(): array
    {
        return Capsule::select("
            SELECT
                DATE(lastlogin) as date,
                COUNT(*) as logins
            FROM tblclients
            WHERE lastlogin >= DATE_SUB(NOW(), INTERVAL 30 DAY)
            GROUP BY DATE(lastlogin)
        ");
    }

    private function getCohortData(): array
    {
        // Monthly cohort analysis
        return Capsule::select("
            SELECT
                DATE_FORMAT(datecreated, '%Y-%m') as cohort,
                COUNT(*) as clients,
                SUM(CASE WHEN DATEDIFF(NOW(), datecreated) <= 30 THEN 1 ELSE 0 END) as retained_30d,
                SUM(CASE WHEN DATEDIFF(NOW(), datecreated) <= 90 THEN 1 ELSE 0 END) as retained_90d
            FROM tblclients
            WHERE datecreated >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
            GROUP BY DATE_FORMAT(datecreated, '%Y-%m')
        ");
    }

    private function getEngagementMetrics(): array
    {
        return [
            'avg_lifetime' => Capsule::select("
                SELECT AVG(DATEDIFF(NOW(), datecreated)) as avg_days
                FROM tblclients
            ")[0]->avg_days ?? 0,
            'avg_services_per_client' => Capsule::select("
                SELECT AVG(services) as avg
                FROM (
                    SELECT userid, COUNT(*) as services
                    FROM tblhosting
                    GROUP BY userid
                ) as client_services
            ")[0]->avg ?? 0
        ];
    }
}
```

## Related Workflows
- [WHMCS Conversion Report](./whmcs-conversion-report.md)
- [WHMCS Retention Report](./whmcs-retention-report.md)