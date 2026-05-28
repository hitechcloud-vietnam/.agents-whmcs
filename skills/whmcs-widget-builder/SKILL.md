# WHMCS Widget Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building dashboard widgets for WHMCS admin area.

## When to Use

- Creating custom admin dashboard widgets
- Building analytics/reporting widgets
- Displaying module-specific statistics

## Widget Patterns

### Widget Class

```php
<?php
// modules/widgets/{WidgetName}.php

use WHMCS\Module\Contracts\WidgetModuleInterface;

class {WidgetName} implements WidgetModuleInterface {
    public static function getName(): string {
        return 'Widget Name';
    }

    public static function getDescription(): string {
        return 'Widget description';
    }

    public static function getSize(): string {
        return 'one-third'; // one-third, one-half, two-thirds, full
    }

    public function getData(): array {
        // Fetch widget data
        $stats = Capsule::table('mod_mymodule_data')
            ->selectRaw('COUNT(*) as total, SUM(amount) as revenue')
            ->first();

        return [
            'total' => $stats->total ?? 0,
            'revenue' => $stats->revenue ?? 0,
        ];
    }

    public function generateOutput(array $data): string {
        return <<<HTML
<div class="widget">
    <div class="widget-header">
        <div class="widget-title">Title</div>
    </div>
    <div class="widget-content">
        <div class="stat">
            <div class="stat-value">{$data['total']}</div>
            <div class="stat-label">Total Items</div>
        </div>
        <div class="stat">
            <div class="stat-value">{$data['revenue']}</div>
            <div class="stat-label">Revenue</div>
        </div>
    </div>
</div>
HTML;
    }
}
```

### Widget Registration

```php
// hooks.php or within addon module

add_hook('AdminHomepage', 1, function() {
    // Register widget
    return [
        'name' => '{WidgetName}',
        'filename' => '{WidgetName}',
        'title' => 'Widget Title',
        'description' => 'Description',
        'size' => 'one-third',
    ];
});
```

### Statistics Widget

```php
public function getData(): array {
    return [
        'pending_orders' => Capsule::table('tblhosting')
            ->where('domainstatus', 'Pending')
            ->count(),

        'active_services' => Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count(),

        'monthly_revenue' => Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereMonth('datepaid', date('m'))
            ->sum('total'),

        'tickets_open' => Capsule::table('tbltickets')
            ->where('status', 'Open')
            ->count(),
    ];
}
```

---

**Related Skills:**
- whmcs-admin-ui-builder
- whmcs-reporting
- whmcs-addon-builder
