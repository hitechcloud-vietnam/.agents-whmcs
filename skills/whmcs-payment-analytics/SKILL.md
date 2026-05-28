# WHMCS Payment Analytics Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building payment analytics and reconciliation.

## When to Use

- Payment reporting modules
- Reconciliation tools
- Revenue dashboards

## Payment Analytics Patterns

```php
<?php
class PaymentAnalytics {
    public function getDailyRevenue(\DateTime $date): array {
        return Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereDate('datepaid', $date->format('Y-m-d'))
            ->selectRaw('
                SUM(subtotal) as subtotal,
                SUM(tax) as tax,
                SUM(total) as total
            ')
            ->first();
    }

    public function getRevenueByGateway(): array {
        return Capsule::table('tbltransfers')
            ->join('tblpaymentgateways', 'tbltransfers.gateway', '=', 'tblpaymentgateways.gateway')
            ->where('tbltransfers.status', 'Completed')
            ->selectRaw('
                tbltransfers.gateway,
                tblpaymentgateways.name,
                COUNT(*) as count,
                SUM(tbltransfers.amount) as total
            ')
            ->groupBy('tbltransfers.gateway')
            ->get();
    }

    public function getFailedPayments(): array {
        return Capsule::table('tbltransfers')
            ->where('status', 'Failed')
            ->whereDate('created_at', '>=', date('Y-m-d', strtotime('-7 days')))
            ->get();
    }

    public function getPendingPayouts(): array {
        return Capsule::table('mod_affiliate_payouts')
            ->where('status', 'pending')
            ->selectRaw('
                COUNT(*) as count,
                SUM(amount) as total
            ')
            ->first();
    }

    public function getRevenueTrend(int $days = 30): array {
        $data = Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereDate('datepaid', '>=', date('Y-m-d', strtotime("-$days days")))
            ->selectRaw("DATE(datepaid) as date, SUM(total) as revenue")
            ->groupBy('date')
            ->get();

        return $data->keyBy('date');
    }

    public function getAverageOrderValue(): float {
        return Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereDate('datepaid', '>=', date('Y-m-d', strtotime('-30 days')))
            ->avg('total') ?? 0;
    }
}
```

---

**Related Skills:**
- whmcs-reporting
- whmcs-gateway-builder
- whmcs-admin-ui-builder
