# WHMCS Reporting Workflow

## Overview
This workflow establishes comprehensive reporting for business intelligence and decision-making.

## Step 1: Reporting Service

```php
<?php
// src/Service/ReportingService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class ReportingService
{
    public function getMonthlyRecurringRevenue(array $filters = []): array
    {
        $query = Capsule::table('tblhosting')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->join('tblpricing', 'tblproducts.id', '=', 'tblpricing.relid')
            ->where('tblhosting.domainstatus', 'Active')
            ->where('tblpricing.type', 'product');

        $services = $query->get();

        $revenue = [
            'monthly' => 0,
            'quarterly' => 0,
            'semiannually' => 0,
            'annually' => 0,
            'total_mrr' => 0
        ];

        foreach ($services as $service) {
            $cycle = strtolower($service->billingcycle);
            $amount = (float)($service->{$cycle} ?? $service->monthly ?? 0);

            $revenue[$cycle] += $amount;

            // Convert to monthly
            $revenue['total_mrr'] += match ($cycle) {
                'monthly' => $amount,
                'quarterly' => $amount / 3,
                'semiannually' => $amount / 6,
                'annually' => $amount / 12,
                default => $amount
            };
        }

        return $revenue;
    }

    public function getClientRetentionReport(): array
    {
        $totalClients = Capsule::table('tblclients')->count();
        $activeClients = Capsule::table('tblclients')->where('status', 'Active')->count();
        $inactiveClients = Capsule::table('tblclients')->where('status', '!=', 'Active')->count();

        $cancelledThisMonth = Capsule::table('tblhosting')
            ->where('domainstatus', 'Cancelled')
            ->where('cancellation_date', '>=', date('Y-m-01'))
            ->count();

        $churnRate = $totalClients > 0
            ? round(($cancelledThisMonth / $totalClients) * 100, 2)
            : 0;

        return [
            'total_clients' => $totalClients,
            'active_clients' => $activeClients,
            'inactive_clients' => $inactiveClients,
            'cancelled_this_month' => $cancelledThisMonth,
            'churn_rate' => $churnRate,
            'retention_rate' => 100 - $churnRate
        ];
    }

    public function getServiceHealthReport(): array
    {
        $statusCounts = Capsule::table('tblhosting')
            ->select('domainstatus')
            ->selectRaw('COUNT(*) as count')
            ->groupBy('domainstatus')
            ->get()
            ->keyBy('domainstatus')
            ->map(fn($r) => $r->count);

        return [
            'active' => $statusCounts['Active'] ?? 0,
            'suspended' => $statusCounts['Suspended'] ?? 0,
            'cancelled' => $statusCounts['Cancelled'] ?? 0,
            'terminated' => $statusCounts['Terminated'] ?? 0,
            'pending' => $statusCounts['Pending'] ?? 0,
            'total' => $statusCounts->sum() ?? 0
        ];
    }

    public function getInvoiceSummary(string $startDate, string $endDate): array
    {
        $invoices = Capsule::table('tblinvoices')
            ->whereBetween('date', [$startDate, $endDate])
            ->get();

        $summary = [
            'total_invoices' => $invoices->count(),
            'paid' => 0,
            'unpaid' => 0,
            'cancelled' => 0,
            'total_amount' => 0,
            'paid_amount' => 0,
            'unpaid_amount' => 0
        ];

        foreach ($invoices as $invoice) {
            $summary['total_amount'] += $invoice->total;

            switch ($invoice->status) {
                case 'Paid':
                    $summary['paid']++;
                    $summary['paid_amount'] += $invoice->total;
                    break;
                case 'Unpaid':
                    $summary['unpaid']++;
                    $summary['unpaid_amount'] += $invoice->total;
                    break;
                case 'Cancelled':
                    $summary['cancelled']++;
                    break;
            }
        }

        return $summary;
    }

    public function getTopProducts(int $limit = 10): array
    {
        return Capsule::table('tblhosting')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->select('tblproducts.name')
            ->selectRaw('COUNT(*) as count')
            ->where('tblhosting.domainstatus', 'Active')
            ->groupBy('tblproducts.id')
            ->orderBy('count', 'desc')
            ->limit($limit)
            ->get();
    }

    public function getPaymentMethodReport(): array
    {
        return Capsule::table('tblaccounts')
            ->select('paymentmethod')
            ->selectRaw('COUNT(*) as count')
            ->selectRaw('SUM(amountin) as total')
            ->where('amountin', '>', 0)
            ->groupBy('paymentmethod')
            ->get();
    }
}
```

## Step 2: Admin Report Widget

```php
<?php
// admin/report_widget.php

add_hook('AdminHomepage', 1, function() {
    $reportService = new \WHMCS\Module\Addon\YourModule\Service\ReportingService();

    return [
        'templatefile' => 'admin/widgets/reports',
        'vars' => [
            'mrr_report' => $reportService->getMonthlyRecurringRevenue(),
            'retention_report' => $reportService->getClientRetentionReport(),
            'health_report' => $reportService->getServiceHealthReport()
        ]
    ];
});
```

## Verification Checklist

- [ ] Reporting service implemented
- [ ] MRR calculation correct
- [ ] Retention report working
- [ ] Service health report working
- [ ] Invoice summary working
- [ ] Admin widget configured
