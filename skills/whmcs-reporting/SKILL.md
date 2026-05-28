# WHMCS Reporting Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building reporting and analytics for WHMCS modules.

## When to Use

- Creating admin dashboards
- Generating usage reports
- Building analytics modules

## Reporting Patterns

### Statistics Collection

```php
function collectStats(int $userId): array {
    return [
        'total_services' => Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->count(),

        'active_services' => Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->where('domainstatus', 'Active')
            ->count(),

        'total_spent' => Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->where('status', 'Paid')
            ->sum('total'),

        'pending_invoices' => Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->where('status', 'Unpaid')
            ->count(),

        'open_tickets' => Capsule::table('tbltickets')
            ->where('userid', $userId)
            ->where('status', 'Open')
            ->count(),
    ];
}
```

### Report Generation

```php
function generateMonthlyReport(string $yearMonth): array {
    $parts = explode('-', $yearMonth);
    $year = $parts[0];
    $month = $parts[1];

    $startDate = date('Y-m-01', strtotime("{$year}-{$month}-01"));
    $endDate = date('Y-m-t', strtotime("{$year}-{$month}-01"));

    return [
        'period' => $yearMonth,
        'new_orders' => countNewOrders($startDate, $endDate),
        'revenue' => calculateRevenue($startDate, $endDate),
        'top_products' => getTopProducts($startDate, $endDate),
        'churn_rate' => calculateChurnRate($startDate, $endDate),
    ];
}

function countNewOrders(string $start, string $end): int {
    return Capsule::table('tblorders')
        ->whereBetween('datecreated', [$start, $end])
        ->where('status', 'Active')
        ->count();
}

function calculateRevenue(string $start, string $end): float {
    return Capsule::table('tblinvoices')
        ->whereBetween('datepaid', [$start, $end])
        ->where('status', 'Paid')
        ->sum('total') ?? 0;
}
```

### Report Display

```smarty
{* templates/admin/report.tpl *}
<div class="report-container">
    <h2>Monthly Report - {$report.period}</h2>

    <div class="stats-grid">
        <div class="stat-card">
            <span class="stat-label">New Orders</span>
            <span class="stat-value">{$report.new_orders}</span>
        </div>
        <div class="stat-card">
            <span class="stat-label">Revenue</span>
            <span class="stat-value">{$report.revenue|formatCurrency}</span>
        </div>
        <div class="stat-card">
            <span class="stat-label">Churn Rate</span>
            <span class="stat-value">{$report.churn_rate}%</span>
        </div>
    </div>

    <h3>Top Products</h3>
    <table class="data-table">
        <thead>
            <tr>
                <th>Product</th>
                <th>Orders</th>
                <th>Revenue</th>
            </tr>
        </thead>
        <tbody>
            {foreach $report.top_products as $product}
            <tr>
                <td>{$product.name|escape:'html'}</td>
                <td>{$product.orders}</td>
                <td>{$product.revenue|formatCurrency}</td>
            </tr>
            {/foreach}
        </tbody>
    </table>

    <button onclick="exportReport('{$report.period}')" class="btn btn-primary">
        Export CSV
    </button>
</div>
```

## Export Functions

```php
function exportReportCsv(array $data, string $filename): void {
    header('Content-Type: text/csv');
    header('Content-Disposition: attachment; filename="' . $filename . '"');

    $output = fopen('php://output', 'w');

    // Header
    fputcsv($output, array_keys($data[0] ?? []));

    // Data
    foreach ($data as $row) {
        fputcsv($output, array_values($row));
    }

    fclose($output);
}
```

## Checklist

- [ ] Statistics collection
- [ ] Report generation
- [ ] Template display
- [ ] CSV export
- [ ] Date range filtering

---

**Related Skills:**
- whmcs-database-design
- whmcs-template-styling
- whmcs-cron-automation