# WHMCS Analytics Integration

Complete guide for analytics and business intelligence integrations.

## Overview

Connect WHMCS with analytics platforms for insights and reporting.

## Business Intelligence

### Metrics Collection

```php
<?php
/**
 * Business metrics collector
 */
class BusinessMetricsCollector
{
    /**
     * Collect all metrics
     */
    public function collect(): array
    {
        return [
            'revenue' => $this->collectRevenueMetrics(),
            'customers' => $this->collectCustomerMetrics(),
            'services' => $this->collectServiceMetrics(),
            'invoices' => $this->collectInvoiceMetrics(),
            'tickets' => $this->collectTicketMetrics(),
        ];
    }
    
    /**
     * Revenue metrics
     */
    private function collectRevenueMetrics(): array
    {
        $today = date('Y-m-d');
        $monthStart = date('Y-m-01');
        $yearStart = date('Y-01-01');
        
        return [
            'today' => $this->getRevenue($today, $today),
            'this_month' => $this->getRevenue($monthStart, $today),
            'this_year' => $this->getRevenue($yearStart, $today),
            'mrr' => $this->calculateMRR(),
            'arr' => $this->calculateARR(),
        ];
    }
    
    /**
     * Get revenue for period
     */
    private function getRevenue(string $start, string $end): float
    {
        return (float) Capsule::table('tblinvoices')
            ->whereBetween('date', [$start, $end])
            ->where('status', 'Paid')
            ->sum('total');
    }
    
    /**
     * Calculate Monthly Recurring Revenue
     */
    private function calculateMRR(): float
    {
        $activeServices = Capsule::table('tblhosting')
            ->whereIn('billingcycle', ['Monthly', 'Quarterly', 'Annually'])
            ->where('domainstatus', 'Active')
            ->get();
        
        $mrr = 0;
        foreach ($activeServices as $service) {
            $mrr += match($service->billingcycle) {
                'Monthly' => $service->amount,
                'Quarterly' => $service->amount / 3,
                'Annually' => $service->amount / 12,
                default => 0,
            };
        }
        
        return $mrr;
    }
    
    /**
     * Calculate Annual Recurring Revenue
     */
    private function calculateARR(): float
    {
        return $this->calculateMRR() * 12;
    }
    
    /**
     * Customer metrics
     */
    private function collectCustomerMetrics(): array
    {
        $today = date('Y-m-d');
        
        return [
            'total' => Capsule::table('tblclients')->count(),
            'active' => Capsule::table('tblclients')->where('status', 'Active')->count(),
            'inactive' => Capsule::table('tblclients')->where('status', 'Inactive')->count(),
            'new_today' => Capsule::table('tblclients')
                ->where('datecreated', '>=', $today)
                ->count(),
            'new_this_month' => Capsule::table('tblclients')
                ->where('datecreated', '>=', date('Y-m-01'))
                ->count(),
        ];
    }
    
    /**
     * Service metrics
     */
    private function collectServiceMetrics(): array
    {
        return [
            'total' => Capsule::table('tblhosting')->count(),
            'active' => Capsule::table('tblhosting')->where('domainstatus', 'Active')->count(),
            'suspended' => Capsule::table('tblhosting')->where('domainstatus', 'Suspended')->count(),
            'terminated' => Capsule::table('tblhosting')->where('domainstatus', 'Terminated')->count(),
            'pending' => Capsule::table('tblhosting')->where('domainstatus', 'Pending')->count(),
        ];
    }
    
    /**
     * Invoice metrics
     */
    private function collectInvoiceMetrics(): array
    {
        return [
            'total' => Capsule::table('tblinvoices')->count(),
            'paid' => Capsule::table('tblinvoices')->where('status', 'Paid')->count(),
            'unpaid' => Capsule::table('tblinvoices')->where('status', 'Unpaid')->count(),
            'overdue' => Capsule::table('tblinvoices')->where('status', 'Overdue')->count(),
            'collections' => $this->calculateCollectionsRate(),
        ];
    }
    
    /**
     * Calculate collections rate
     */
    private function calculateCollectionsRate(): float
    {
        $totalInvoices = Capsule::table('tblinvoices')
            ->whereIn('status', ['Paid', 'Unpaid', 'Overdue'])
            ->count();
        
        $paidInvoices = Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->count();
        
        return $totalInvoices > 0 ? round(($paidInvoices / $totalInvoices) * 100, 2) : 0;
    }
    
    /**
     * Ticket metrics
     */
    private function collectTicketMetrics(): array
    {
        return [
            'total' => Capsule::table('tbltickets')->count(),
            'open' => Capsule::table('tbltickets')->whereIn('status', ['Open', 'Answered'])->count(),
            'closed' => Capsule::table('tbltickets')->where('status', 'Closed')->count(),
            'avg_response_time' => $this->calculateAvgResponseTime(),
        ];
    }
    
    /**
     * Calculate average response time
     */
    private function calculateAvgResponseTime(): string
    {
        $avgMinutes = Capsule::table('tblticketreplies')
            ->selectRaw('AVG(TIMESTAMPDIFF(MINUTE, 
                (SELECT created_at FROM tblticketreplies r2 WHERE r2.tid = tblticketreplies.tid ORDER BY created_at LIMIT 1),
                tblticketreplies.created_at)) as avg_time')
            ->first()->avg_time ?? 0;
        
        $hours = floor($avgMinutes / 60);
        $minutes = $avgMinutes % 60;
        
        return "{$hours}h {$minutes}m";
    }
}
```

## Data Warehouse

### Data Warehouse Export

```php
<?php
/**
 * Export to data warehouse
 */
class DataWarehouseExporter
{
    private PDO $warehouse;
    
    public function __construct(array $config)
    {
        $this->warehouse = new PDO(
            "mysql:host={$config['host']};dbname={$config['database']}",
            $config['username'],
            $config['password']
        );
    }
    
    /**
     * Export clients dimension
     */
    public function exportClients(): int
    {
        $clients = Capsule::table('tblclients')->get();
        $count = 0;
        
        $stmt = $this->warehouse->prepare('
            INSERT INTO dim_clients 
            (client_id, email, first_name, last_name, company, country, status, created_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            ON DUPLICATE KEY UPDATE
                email = VALUES(email),
                status = VALUES(status)
        ');
        
        foreach ($clients as $client) {
            $stmt->execute([
                $client->id,
                $client->email,
                $client->firstname,
                $client->lastname,
                $client->companyname,
                $client->country,
                $client->status,
                $client->datecreated,
            ]);
            $count++;
        }
        
        return $count;
    }
    
    /**
     * Export services dimension
     */
    public function exportServices(): int
    {
        $services = Capsule::table('tblhosting')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->get();
        
        $count = 0;
        $stmt = $this->warehouse->prepare('
            INSERT INTO dim_services
            (service_id, client_id, product_name, domain, status, amount, billing_cycle, created_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            ON DUPLICATE KEY UPDATE status = VALUES(status)
        ');
        
        foreach ($services as $service) {
            $stmt->execute([
                $service->id,
                $service->userid,
                $service->name,
                $service->domain,
                $service->domainstatus,
                $service->amount,
                $service->billingcycle,
                $service->regdate,
            ]);
            $count++;
        }
        
        return $count;
    }
    
    /**
     * Export invoices fact
     */
    public function exportInvoices(): int
    {
        $invoices = Capsule::table('tblinvoices')
            ->where('status', '!=', 'Draft')
            ->get();
        
        $count = 0;
        $stmt = $this->warehouse->prepare('
            INSERT INTO fact_invoices
            (invoice_id, client_id, total, tax, status, created_at, paid_at)
            VALUES (?, ?, ?, ?, ?, ?, ?)
            ON DUPLICATE KEY UPDATE status = VALUES(status)
        ');
        
        foreach ($invoices as $invoice) {
            $stmt->execute([
                $invoice->id,
                $invoice->userid,
                $invoice->total,
                $invoice->tax,
                $invoice->status,
                $invoice->date,
                $invoice->datepaid,
            ]);
            $count++;
        }
        
        return $count;
    }
}
```

## Cohort Analysis

```php
<?php
/**
 * Cohort analysis
 */
class CohortAnalyzer
{
    /**
     * Generate cohort analysis
     */
    public function analyze(int $cohorts = 12): array
    {
        $results = [];
        
        for ($i = 0; $i < $cohorts; $i++) {
            $cohortDate = date('Y-m-01', strtotime("-{$i} months"));
            $cohortLabel = date('M Y', strtotime($cohortDate));
            
            $results[$cohortLabel] = $this->analyzeCohort($cohortDate);
        }
        
        return $results;
    }
    
    /**
     * Analyze single cohort
     */
    private function analyzeCohort(string $cohortDate): array
    {
        // Get clients who signed up in this month
        $cohortClients = Capsule::table('tblclients')
            ->whereRaw("DATE_FORMAT(datecreated, '%Y-%m') = ?", [date('Y-m', strtotime($cohortDate))])
            ->pluck('id')
            ->toArray();
        
        $totalClients = count($cohortClients);
        if ($totalClients === 0) {
            return ['total' => 0, 'retention' => []];
        }
        
        $retention = [];
        
        // Calculate retention for each month after signup
        for ($month = 0; $month <= 12; $month++) {
            $periodStart = date('Y-m-d', strtotime("+{$month} months", strtotime($cohortDate)));
            $periodEnd = date('Y-m-t', strtotime("+{$month} months", strtotime($cohortDate)));
            
            // Count clients with activity in this period
            $activeClients = Capsule::table('tblinvoices')
                ->whereIn('userid', $cohortClients)
                ->whereBetween('date', [$periodStart, $periodEnd])
                ->where('status', 'Paid')
                ->distinct('userid')
                ->count('userid');
            
            $retention[] = [
                'month' => $month,
                'active' => $activeClients,
                'rate' => round(($activeClients / $totalClients) * 100, 1),
            ];
        }
        
        return [
            'total' => $totalClients,
            'retention' => $retention,
        ];
    }
}
```

## Funnel Analysis

```php
<?php
/**
 * Conversion funnel analysis
 */
class FunnelAnalyzer
{
    /**
     * Analyze order funnel
     */
    public function analyzeOrderFunnel(): array
    {
        $funnel = [];
        
        // Visits (cart sessions)
        $funnel['cart_sessions'] = Capsule::table('mod_cart_sessions')
            ->where('created_at', '>=', date('Y-m-01'))
            ->count();
        
        // Started checkout
        $funnel['checkout_started'] = Capsule::table('mod_cart_sessions')
            ->where('checkout_started', 1)
            ->where('created_at', '>=', date('Y-m-01'))
            ->count();
        
        // Completed order
        $funnel['order_completed'] = Capsule::table('tblorders')
            ->where('status', 'Active')
            ->where('date', '>=', date('Y-m-01'))
            ->count();
        
        // Calculate conversion rates
        $rates = [];
        $keys = array_keys($funnel);
        for ($i = 1; $i < count($keys); $i++) {
            $from = $funnel[$keys[$i - 1]];
            $to = $funnel[$keys[$i]];
            
            $rates[$keys[$i]] = $from > 0 ? round(($to / $from) * 100, 2) : 0;
        }
        
        $overallRate = $funnel['cart_sessions'] > 0 
            ? round(($funnel['order_completed'] / $funnel['cart_sessions']) * 100, 2) 
            : 0;
        
        return [
            'stages' => $funnel,
            'conversion_rates' => $rates,
            'overall_conversion' => $overallRate,
        ];
    }
    
    /**
     * Analyze signup funnel
     */
    public function analyzeSignupFunnel(): array
    {
        $funnel = [];
        
        // Page visits
        $funnel['page_visits'] = Capsule::table('mod_page_views')
            ->where('page', 'register')
            ->where('created_at', '>=', date('Y-m-01'))
            ->count();
        
        // Form started
        $funnel['form_started'] = Capsule::table('mod_registration_sessions')
            ->where('created_at', '>=', date('Y-m-01'))
            ->count();
        
        // Registration completed
        $funnel['registration_completed'] = Capsule::table('tblclients')
            ->where('datecreated', '>=', date('Y-m-01'))
            ->count();
        
        return [
            'stages' => $funnel,
            'conversion_rate' => $funnel['page_visits'] > 0 
                ? round(($funnel['registration_completed'] / $funnel['page_visits']) * 100, 2) 
                : 0,
        ];
    }
}
```

## Best Practices

1. **Track key metrics** - Focus on actionable metrics
2. **Segment data** - Break down by product, region, etc.
3. **Real-time updates** - Keep dashboards current
4. **Historical comparison** - Compare to previous periods
5. **Anomaly detection** - Alert on unusual patterns
6. **Export capabilities** - Allow data export

## Related Documentation

- [whmcs-integration-reporting.md](whmcs-integration-reporting.md)
- [whmcs-integration-api.md](whmcs-integration-api.md)
