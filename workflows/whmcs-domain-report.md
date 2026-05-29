# WHMCS Domain Report Workflow

## Overview
This workflow generates domain portfolio and expiry reports.

## Step-by-Step Process

```php
<?php
// /includes/reports/DomainReportGenerator.php

class DomainReportGenerator
{
    public function generate(array $params = []): array
    {
        return [
            'summary' => $this->getDomainSummary(),
            'expiring_soon' => $this->getExpiringDomains(),
            'by_registrar' => $this->getByRegistrar(),
            'transfer_status' => $this->getTransferStatus()
        ];
    }

    private function getDomainSummary(): array
    {
        return [
            'total_domains' => Capsule::table('tbldomains')->count(),
            'active' => Capsule::table('tbldomains')->where('status', 'Active')->count(),
            'expired' => Capsule::table('tbldomains')->where('status', 'Expired')->count(),
            'transferred_away' => Capsule::table('tbldomains')->where('status', 'Transferred Away')->count()
        ];
    }

    private function getExpiringDomains(): array
    {
        $warningDate = date('Y-m-d', strtotime('+30 days'));
        $criticalDate = date('Y-m-d', strtotime('+7 days'));

        return Capsule::select("
            SELECT
                domain,
                expirydate,
                DATEDIFF(expirydate, CURDATE()) as days_remaining,
                registrationperiod,
                CASE
                    WHEN DATEDIFF(expirydate, CURDATE()) <= 7 THEN 'critical'
                    WHEN DATEDIFF(expirydate, CURDATE()) <= 30 THEN 'warning'
                    ELSE 'normal'
                END as urgency
            FROM tbldomains
            WHERE status = 'Active'
            AND expirydate <= ?
            AND expirydate >= CURDATE()
            ORDER BY expirydate ASC
        ", [$warningDate]);
    }

    private function getByRegistrar(): array
    {
        return Capsule::select("
            SELECT
                registrar,
                COUNT(*) as domains,
                SUM(CASE WHEN status = 'Active' THEN 1 ELSE 0 END) as active,
                MIN(expirydate) as next_expiry
            FROM tbldomains
            GROUP BY registrar
            ORDER BY domains DESC
        ");
    }

    private function getTransferStatus(): array
    {
        return Capsule::table('tbldomains')
            ->where('status', 'Transfer')
            ->select('domain', 'transfertime', 'status')
            ->get();
    }
}
```

## Related Workflows
- [WHMCS Report Automation](./whmcs-report-automation.md)