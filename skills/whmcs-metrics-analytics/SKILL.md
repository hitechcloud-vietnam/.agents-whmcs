# WHMCS Metrics & Analytics Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building analytics and metrics dashboards.

## When to Use

- Creating business intelligence modules
- Building custom reports
- Tracking module usage

## Analytics Patterns

```php
<?php
class MetricsCollector {
    public function trackEvent(string $event, array $data = []): void {
        Capsule::table('mod_metrics_events')->insert([
            'event' => $event,
            'data' => json_encode($data),
            'user_id' => $_SESSION['uid'] ?? 0,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function getEventStats(string $event, \DateTime $from, \DateTime $to): array {
        return Capsule::table('mod_metrics_events')
            ->selectRaw('DATE(created_at) as date, COUNT(*) as count')
            ->where('event', $event)
            ->whereBetween('created_at', [$from->format('Y-m-d'), $to->format('Y-m-d')])
            ->groupBy('date')
            ->get();
    }

    public function getRevenueMetrics(\DateTime $from, \DateTime $to): array {
        $paidInvoices = Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereBetween('datepaid', [$from->format('Y-m-d'), $to->format('Y-m-d')])
            ->selectRaw('
                COUNT(*) as invoice_count,
                SUM(total) as total_revenue,
                AVG(total) as avg_order
            ')
            ->first();

        return [
            'count' => $paidInvoices->invoice_count ?? 0,
            'revenue' => $paidInvoices->total_revenue ?? 0,
            'avg_order' => $paidInvoices->avg_order ?? 0,
        ];
    }

    public function getServiceMetrics(): array {
        return Capsule::table('tblhosting')
            ->selectRaw('domainstatus, COUNT(*) as count')
            ->groupBy('domainstatus')
            ->get()
            ->keyBy('domainstatus');
    }
}
```

### Dashboard Hook
```php
add_hook('AdminHomepage', 1, function() {
    $metrics = new MetricsCollector();

    return [
        'key_metrics_widget' => [
            'display' => true,
            'data' => [
                'active_services' => $metrics->getServiceMetrics()['Active']->count ?? 0,
                'monthly_revenue' => $metrics->getRevenueMetrics(
                    new \DateTime('first day of this month'),
                    new \DateTime()
                )['revenue'],
            ],
        ],
    ];
});
```

---

**Related Skills:**
- whmcs-reporting
- whmcs-admin-ui-builder
- whmcs-widget-builder
