# WHMCS Billing Dashboard Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build billing analytics dashboard modules.

## Billing Dashboard

```php
<?php
class BillingDashboard {
    public function getOverviewMetrics(): array {
        $today = date('Y-m-d');
        $startOfMonth = date('Y-m-01');

        return [
            'today' => $this->getDailyMetrics($today),
            'month' => $this->getMonthlyMetrics($startOfMonth, $today),
            'year' => $this->getYearlyMetrics(),
            'trends' => $this->getTrends(),
        ];
    }

    public function getDailyMetrics(string $date): array {
        $paid = Capsule::table('tblinvoices')
            ->where('date', $date)
            ->where('status', 'Paid')
            ->selectRaw('COUNT(*) as count, SUM(total) as total')
            ->first();

        $outstanding = Capsule::table('tblinvoices')
            ->whereIn('status', ['Unpaid', 'Overdue'])
            ->where('duedate', $date)
            ->selectRaw('COUNT(*) as count, SUM(total) as total')
            ->first();

        return [
            'revenue' => $paid->total ?? 0,
            'invoice_count' => $paid->count ?? 0,
            'outstanding_count' => $outstanding->count ?? 0,
            'outstanding_amount' => $outstanding->total ?? 0,
        ];
    }

    public function getMonthlyMetrics(string $startDate, string $endDate): array {
        $paid = Capsule::table('tblinvoices')
            ->whereBetween('date', [$startDate, $endDate])
            ->where('status', 'Paid')
            ->selectRaw('SUM(total) as total, COUNT(*) as count')
            ->first();

        $projected = Capsule::table('tblhosting')
            ->where('billingcycle', 'Monthly')
            ->selectRaw('SUM(regdate) as mrr')
            ->first();

        return [
            'revenue' => $paid->total ?? 0,
            'invoice_count' => $paid->count ?? 0,
            'mrr' => $projected->mrr ?? 0,
            'growth' => $this->calculateGrowth($startDate, $endDate),
        ];
    }

    public function getTopClients(int $limit = 10): array {
        return Capsule::table('tblinvoices')
            ->join('tblclients', 'tblinvoices.userid', '=', 'tblclients.id')
            ->where('tblinvoices.status', 'Paid')
            ->where('tblinvoices.date', '>=', date('Y-m-d', strtotime('-1 year')))
            ->selectRaw('
                tblclients.id,
                tblclients.firstname,
                tblclients.lastname,
                tblclients.email,
                SUM(tblinvoices.total) as total_revenue,
                COUNT(tblinvoices.id) as invoice_count
            ')
            ->groupBy('tblclients.id')
            ->orderBy('total_revenue', 'desc')
            ->limit($limit)
            ->get();
    }

    public function getRevenueForecast(int $months = 6): array {
        $forecast = [];

        for ($i = 0; $i < $months; $i++) {
            $month = date('Y-m', strtotime("+{$i} months"));

            $existing = Capsule::table('tblhosting')
                ->whereIn('billingcycle', ['Monthly', 'Quarterly', 'Semi-Annually'])
                ->selectRaw('SUM(tblproducts.recurring) as total')
                ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
                ->where('tblhosting.domainstatus', 'Active')
                ->first();

            $cancellations = $this->estimateCancellations($month);

            $forecast[] = [
                'month' => $month,
                'projected' => $existing->total ?? 0,
                'churn' => $cancellations,
                'net' => ($existing->total ?? 0) - $cancellations,
            ];
        }

        return $forecast;
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-reporting