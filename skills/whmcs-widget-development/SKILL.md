# WHMCS Widget Development Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for developing custom dashboard widgets for WHMCS admin area and client portal, including statistics widgets, charts, and interactive widget components.

## When to Use

- Creating custom admin dashboard widgets
- Building client portal widgets
- Adding real-time statistics widgets
- Creating interactive widget components
- Extending WHMCS dashboard functionality

## Widget System Overview

### 1. Admin Dashboard Widget Structure

```php
<?php
// includes/widgets/MyWidget.php

namespace WHMCS\Module\Widget;

use WHMCS\Admin\Helper;
use WHMCS\Dashboard\Widgets\AbstractWidget;

class MyWidget extends AbstractWidget {
    protected $title = 'My Custom Widget';
    protected $description = 'Displays custom statistics and data';
    protected $type = 'standard'; // standard, chart, table, info
    protected $cacheTimeout = 5; // Minutes

    /**
     * Widget data and rendering
     */
    public function getData(array $parameters): array {
        // Fetch and return widget data
        return [
            'total_clients' => $this->getTotalClients(),
            'active_services' => $this->getActiveServices(),
            'monthly_revenue' => $this->getMonthlyRevenue(),
            'recent_activity' => $this->getRecentActivity(),
        ];
    }

    /**
     * Render the widget
     */
    public function generateOutput(array $widgetData, array $widgetParameters): string {
        $template = new \WHMCS\Smarty\Frontend();

        $template->assign('data', $widgetData);
        $template->assign('title', $this->title);

        return $template->fetch(__DIR__ . '/templates/widget.tpl');
    }

    /**
     * Widget configuration options
     */
    public function getConfigurableParameters(): array {
        return [
            'display_title' => [
                'type' => 'yesno',
                'default' => true,
                'label' => 'Display Widget Title',
            ],
            'refresh_interval' => [
                'type' => 'dropdown',
                'options' => [
                    '1' => 'Every minute',
                    '5' => 'Every 5 minutes',
                    '15' => 'Every 15 minutes',
                ],
                'default' => '5',
                'label' => 'Refresh Interval',
            ],
            'show_chart' => [
                'type' => 'yesno',
                'default' => true,
                'label' => 'Show Chart',
            ],
        ];
    }

    // Data fetching methods
    private function getTotalClients(): int {
        return \WHMCS\Database\Capsule::table('tblclients')
            ->where('status', '!=', 'Closed')
            ->count();
    }

    private function getActiveServices(): int {
        return \WHMCS\Database\Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count();
    }

    private function getMonthlyRevenue(): float {
        return \WHMCS\Database\Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->where('date', '>=', date('Y-m-01'))
            ->sum('total');
    }

    private function getRecentActivity(): array {
        return \WHMCS\Database\Capsule::table('tblactivitylog')
            ->orderBy('id', 'DESC')
            ->limit(10)
            ->get()
            ->toArray();
    }
}
```

### 2. Widget Registration Hook

```php
<?php
// hooks.php - Register custom widgets

add_hook('AdminAreaHeadOutput', 1, function($vars) {
    // Register widget
    if ($vars['filename'] === 'index.php' && !isset($_GET['nocache'])) {
        \WHMCS\Module\Widget\WidgetManager::registerWidget([
            'name' => 'MyCustomWidget',
            'namespace' => 'WHMCS\\Module\\Widget\\MyWidget',
            'title' => 'My Widget',
            'description' => 'Custom dashboard widget',
            'size' => 'col-md-4', // Bootstrap grid class
        ]);
    }
});
```

## Widget Template Patterns

### 1. Basic Stats Widget

```html
<!-- templates/widgets/stats_widget.tpl -->

<div class="widget {$widget.class|default:'col-md-4'}">
    <div class="widget-heading">
        <h3 class="widget-title">{$title}</h3>
        <div class="widget-actions">
            <a href="#" class="btn btn-xs btn-default" data-widget-refresh>
                <i class="fa fa-refresh"></i>
            </a>
        </div>
    </div>
    <div class="widget-body">
        <div class="row">
            <div class="col-xs-6">
                <div class="stat-value">{$data.total_clients}</div>
                <div class="stat-label">Total Clients</div>
            </div>
            <div class="col-xs-6">
                <div class="stat-value text-success">{$data.active_services}</div>
                <div class="stat-label">Active Services</div>
            </div>
        </div>
        <div class="row">
            <div class="col-xs-12">
                <div class="revenue-display">
                    <span class="label">Monthly Revenue:</span>
                    <span class="value">{$data.monthly_revenue|currency_format}</span>
                </div>
            </div>
        </div>
    </div>
</div>
```

### 2. Chart Widget (Line/Bar Chart)

```html
<!-- templates/widgets/chart_widget.tpl -->

<div class="widget chart-widget">
    <div class="widget-heading">
        <h3 class="widget-title">{$title}</h3>
        <div class="widget-actions">
            <div class="btn-group">
                <button type="button" class="btn btn-xs btn-default dropdown-toggle"
                        data-toggle="dropdown">
                    <i class="fa fa-ellipsis-v"></i>
                </button>
                <ul class="dropdown-menu dropdown-menu-right">
                    <li><a href="#" data-period="week">Last 7 Days</a></li>
                    <li><a href="#" data-period="month">Last 30 Days</a></li>
                    <li><a href="#" data-period="year">Last 12 Months</a></li>
                </ul>
            </div>
        </div>
    </div>
    <div class="widget-body">
        <canvas id="widgetChart" height="200"></canvas>
        <div class="chart-legend">
            {foreach $data.legend as $item}
            <span class="legend-item">
                <span class="legend-color" style="background: {$item.color}"></span>
                {$item.label}
            </span>
            {/foreach}
        </div>
    </div>

    <script>
    (function() {
        var ctx = document.getElementById('widgetChart').getContext('2d');
        var chart = new Chart(ctx, {
            type: '{$chart_type|default:"line"}',
            data: {
                labels: {$data.labels|json_encode},
                datasets: {$data.datasets|json_encode}
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                    legend: { display: false }
                },
                scales: {
                    y: {
                        beginAtZero: true,
                        grid: { color: '#f0f0f0' }
                    },
                    x: {
                        grid: { display: false }
                    }
                }
            }
        });

        // Refresh handler
        document.querySelector('[data-widget-refresh]').addEventListener('click', function(e) {
            e.preventDefault();
            // AJAX refresh logic
        });
    })();
    </script>
</div>
```

### 3. Table/List Widget

```html
<!-- templates/widgets/table_widget.tpl -->

<div class="widget table-widget">
    <div class="widget-heading">
        <h3 class="widget-title">{$title}</h3>
        <a href="{$widget.link|default:'#'}" class="btn btn-xs btn-link">
            View All <i class="fa fa-arrow-right"></i>
        </a>
    </div>
    <div class="widget-body no-padding">
        <table class="table table-striped table-condensed">
            <thead>
                <tr>
                    <th>{$data.columns.name}</th>
                    <th>{$data.columns.status}</th>
                    <th class="text-right">{$data.columns.amount}</th>
                </tr>
            </thead>
            <tbody>
                {foreach $data.rows as $row}
                <tr>
                    <td>
                        <a href="{$row.link}">{$row.name}</a>
                    </td>
                    <td>
                        <span class="label label-{$row.status_class}">
                            {$row.status}
                        </span>
                    </td>
                    <td class="text-right">
                        {$row.amount|currency_format}
                    </td>
                </tr>
                {foreachelse}
                <tr>
                    <td colspan="3" class="text-center text-muted">
                        No data available
                    </td>
                </tr>
                {/foreach}
            </tbody>
        </table>
    </div>
</div>
```

## Advanced Widget Patterns

### 1. Real-Time Stats Widget

```php
<?php
// modules/addons/myaddon/widgets/RealtimeStatsWidget.php

namespace WHMCS\Module\Addon\MyAddon\Widgets;

class RealtimeStatsWidget {
    protected $title = 'Real-Time Statistics';
    protected $cacheTimeout = 1; // 1 minute

    public function getData(): array {
        return [
            'online_users' => $this->getOnlineUsers(),
            'active_tickets' => $this->getActiveTickets(),
            'pending_orders' => $this->getPendingOrders(),
            'server_load' => $this->getServerLoad(),
            'alerts' => $this->getAlerts(),
        ];
    }

    public function getAjaxResponse(): void {
        header('Content-Type: application/json');
        echo json_encode($this->getData());
    }

    private function getOnlineUsers(): int {
        $timeout = 900; // 15 minutes

        return \WHMCS\Database\Capsule::table('tblsessions')
            ->where('lastvisit', '>', date('Y-m-d H:i:s', time() - $timeout))
            ->count();
    }

    private function getActiveTickets(): int {
        return \WHMCS\Database\Capsule::table('tbltickets')
            ->whereIn('status', ['Open', 'Answered'])
            ->count();
    }

    private function getPendingOrders(): int {
        return \WHMCS\Database\Capsule::table('tblorders')
            ->where('status', 'Pending')
            ->count();
    }

    private function getServerLoad(): array {
        $servers = \WHMCS\Database\Capsule::table('tblservers')
            ->where('active', 1)
            ->where('disabled', 0)
            ->get();

        $loads = [];
        foreach ($servers as $server) {
            $loads[] = [
                'name' => $server->name,
                'load' => $this->fetchServerLoad($server),
            ];
        }

        return $loads;
    }

    private function fetchServerLoad(object $server): float {
        // Query server for load average via module
        // Placeholder implementation
        return rand(0, 100) / 100 * 2; // Random 0-2 load
    }

    private function getAlerts(): array {
        $alerts = [];

        // Check overdue invoices
        $overdue = \WHMCS\Database\Capsule::table('tblinvoices')
            ->where('status', 'Unpaid')
            ->where('duedate', '<', date('Y-m-d'))
            ->count();

        if ($overdue > 0) {
            $alerts[] = [
                'type' => 'warning',
                'message' => "{$overdue} overdue invoices",
                'link' => 'invoices.php?status=overdue',
            ];
        }

        // Check expiring domains
        $expiring = \WHMCS\Database\Capsule::table('tbldomains')
            ->where('status', 'Active')
            ->where('expirydate', '<=', date('Y-m-d', strtotime('+7 days')))
            ->count();

        if ($expiring > 0) {
            $alerts[] = [
                'type' => 'info',
                'message' => "{$expiring} domains expiring soon",
                'link' => 'domains.php?status=expiring',
            ];
        }

        return $alerts;
    }
}
```

### 2. Widget with AJAX Refresh

```php
<?php
// modules/addons/myaddon/widgets/AjaxWidget.php

namespace WHMCS\Module\Addon\MyAddon\Widgets;

class AjaxWidget {
    protected $title = 'Ajax Data Widget';

    public function getHtml(): string {
        $data = $this->getData();

        $html = '<div class="widget ajax-widget" data-widget-id="' . $this->getId() . '">';
        $html .= '<div class="widget-heading"><h3>' . $this->title . '</h3></div>';
        $html .= '<div class="widget-body">';
        $html .= '<div class="data-container">';

        foreach ($data['items'] as $item) {
            $html .= '<div class="data-item">';
            $html .= '<span class="data-label">' . $item['label'] . '</span>';
            $html .= '<span class="data-value">' . $item['value'] . '</span>';
            $html .= '</div>';
        }

        $html .= '</div></div>';
        $html .= '<div class="widget-footer">';
        $html .= '<span class="last-updated">Last updated: ' . date('H:i:s') . '</span>';
        $html .= '<button class="btn btn-xs btn-refresh"><i class="fa fa-refresh"></i></button>';
        $html .= '</div>';
        $html .= '</div>';

        return $html;
    }

    public function getAjaxData(): array {
        return $this->getData();
    }

    private function getData(): array {
        return [
            'timestamp' => date('Y-m-d H:i:s'),
            'items' => [
                ['label' => 'Total Users', 'value' => 1234],
                ['label' => 'Active Sessions', 'value' => 56],
                ['label' => 'Queue Size', 'value' => 12],
            ],
        ];
    }

    private function getId(): string {
        return 'ajax-widget-' . substr(md5($this->title), 0, 8);
    }
}

// JavaScript for AJAX refresh
$ajaxRefreshScript = <<<JS
<script>
(function() {
    document.querySelectorAll('.ajax-widget').forEach(function(widget) {
        var refreshBtn = widget.querySelector('.btn-refresh');

        refreshBtn.addEventListener('click', function() {
            var widgetId = widget.dataset.widgetId;

            fetch('ajax.php?widget=' + widgetId)
                .then(function(response) { return response.json(); })
                .then(function(data) {
                    updateWidget(widget, data);
                });
        });

        // Auto-refresh every 60 seconds
        setInterval(function() {
            refreshBtn.click();
        }, 60000);
    });

    function updateWidget(widget, data) {
        var container = widget.querySelector('.data-container');
        var items = data.items;

        container.innerHTML = items.map(function(item) {
            return '<div class="data-item">' +
                   '<span class="data-label">' + item.label + '</span>' +
                   '<span class="data-value">' + item.value + '</span>' +
                   '</div>';
        }).join('');

        widget.querySelector('.last-updated').textContent = 'Last updated: ' + data.timestamp;
    }
})();
</script>
JS;
```

### 3. Drill-Down Widget

```php
<?php
// modules/addons/myaddon/widgets/DrilldownWidget.php

namespace WHMCS\Module\Addon\MyAddon\Widgets;

class DrilldownWidget {
    protected $title = 'Service Overview';

    public function getHtml(string $level = 'summary', ?int $parentId = null): string {
        $data = $this->getData($level, $parentId);

        $html = '<div class="widget drilldown-widget">';
        $html .= '<div class="widget-heading">';
        $html .= '<h3>' . $this->title . '</h3>';

        if ($level !== 'summary') {
            $html .= '<a href="#" class="btn btn-xs btn-back" data-level="' . $level . '" data-parent="' . $parentId . '">';
            $html .= '<i class="fa fa-arrow-left"></i> Back';
            $html .= '</a>';
        }

        $html .= '</div>';
        $html .= '<div class="widget-body">';
        $html .= $this->renderLevel($level, $data);
        $html .= '</div></div>';

        return $html;
    }

    private function getData(string $level, ?int $parentId): array {
        switch ($level) {
            case 'summary':
                return $this->getSummaryData();
            case 'services':
                return $this->getServicesData();
            case 'service':
                return $this->getServiceDetailData($parentId);
            default:
                return [];
        }
    }

    private function getSummaryData(): array {
        $stats = [
            ['type' => 'hosting', 'label' => 'Hosting', 'count' => 150, 'icon' => 'fa-server'],
            ['type' => 'domains', 'label' => 'Domains', 'count' => 320, 'icon' => 'fa-globe'],
            ['type' => 'ssl', 'label' => 'SSL Certs', 'count' => 45, 'icon' => 'fa-lock'],
            ['type' => 'other', 'label' => 'Other', 'count' => 25, 'icon' => 'fa-cube'],
        ];

        return ['items' => $stats];
    }

    private function getServicesData(): array {
        $services = \WHMCS\Database\Capsule::table('tblhosting as h')
            ->join('tblproducts as p', 'h.pid', '=', 'p.id')
            ->where('h.domainstatus', 'Active')
            ->select('h.id', 'h.domain', 'p.name as product')
            ->limit(20)
            ->get();

        return ['items' => $services->toArray()];
    }

    private function getServiceDetailData(int $serviceId): array {
        $service = \WHMCS\Database\Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();

        $invoices = \WHMCS\Database\Capsule::table('tblinvoiceitems')
            ->where('relid', $serviceId)
            ->where('type', 'Hosting')
            ->join('tblinvoices', 'tblinvoices.id', '=', 'tblinvoiceitems.invoiceid')
            ->select('tblinvoices.*')
            ->limit(10)
            ->get();

        return [
            'service' => $service,
            'invoices' => $invoices->toArray(),
        ];
    }

    private function renderLevel(string $level, array $data): string {
        switch ($level) {
            case 'summary':
                return $this->renderSummary($data);
            case 'services':
                return $this->renderServiceList($data);
            case 'service':
                return $this->renderServiceDetail($data);
            default:
                return '<p>No data available</p>';
        }
    }

    private function renderSummary(array $data): string {
        $html = '<div class="drilldown-grid">';

        foreach ($data['items'] as $item) {
            $html .= '<a href="#" class="drilldown-item" data-level="services" data-type="' . $item['type'] . '">';
            $html .= '<i class="fa ' . $item['icon'] . '"></i>';
            $html .= '<span class="count">' . $item['count'] . '</span>';
            $html .= '<span class="label">' . $item['label'] . '</span>';
            $html .= '</a>';
        }

        $html .= '</div>';
        return $html;
    }

    private function renderServiceList(array $data): string {
        $html = '<table class="table table-striped">';
        $html .= '<thead><tr><th>Domain</th><th>Product</th><th></th></tr></thead>';
        $html .= '<tbody>';

        foreach ($data['items'] as $item) {
            $html .= '<tr>';
            $html .= '<td>' . $item->domain . '</td>';
            $html .= '<td>' . $item->product . '</td>';
            $html .= '<td><a href="#" class="btn btn-xs btn-default" data-level="service" data-parent="' . $item->id . '">Details</a></td>';
            $html .= '</tr>';
        }

        $html .= '</tbody></table>';
        return $html;
    }

    private function renderServiceDetail(array $data): string {
        $service = $data['service'];

        $html = '<div class="service-detail">';
        $html .= '<h4>' . $service->domain . '</h4>';
        $html .= '<dl class="dl-horizontal">';
        $html .= '<dt>Username:</dt><dd>' . $service->username . '</dd>';
        $html .= '<dt>Status:</dt><dd>' . $service->domainstatus . '</dd>';
        $html .= '<dt>Next Due:</dt><dd>' . $service->nextduedate . '</dd>';
        $html .= '</dl>';

        if (!empty($data['invoices'])) {
            $html .= '<h5>Recent Invoices</h5>';
            $html .= '<ul>';
            foreach ($data['invoices'] as $invoice) {
                $html .= '<li>Invoice #' . $invoice->id . ' - ' . $invoice->total . '</li>';
            }
            $html .= '</ul>';
        }

        $html .= '</div>';
        return $html;
    }
}
```

## Widget System Integration

### 1. Widget Manager Class

```php
<?php
// modules/addons/myaddon/WidgetManager.php

namespace MyAddon;

use WHMCS\Database\Capsule;

class WidgetManager {
    private array $widgets = [];
    private string $location;

    public function __construct(string $location = 'admin') {
        $this->location = $location;
        $this->loadWidgets();
    }

    private function loadWidgets(): void {
        $savedWidgets = $this->getSavedWidgetConfig();

        foreach ($savedWidgets as $widgetConfig) {
            $this->widgets[$widgetConfig['id']] = $widgetConfig;
        }
    }

    public function registerWidget(string $id, array $config): void {
        $this->widgets[$id] = array_merge($config, [
            'id' => $id,
            'location' => $this->location,
            'enabled' => true,
        ]);

        $this->saveWidgetConfig();
    }

    public function unregisterWidget(string $id): void {
        unset($this->widgets[$id]);
        $this->saveWidgetConfig();
    }

    public function getWidgets(): array {
        return $this->widgets;
    }

    public function getWidget(string $id): ?array {
        return $this->widgets[$id] ?? null;
    }

    public function renderWidgets(): string {
        $html = '<div class="widget-container row">';

        foreach ($this->widgets as $widget) {
            if (!$widget['enabled']) {
                continue;
            }

            $widgetClass = $widget['class'] ?? 'GenericWidget';
            $widgetInstance = new $widgetClass($widget);

            $html .= '<div class="' . ($widget['size'] ?? 'col-md-6') . '">';
            $html .= $widgetInstance->render();
            $html .= '</div>';
        }

        $html .= '</div>';
        return $html;
    }

    private function getSavedWidgetConfig(): array {
        $config = Capsule::table('mod_myaddon_widget_config')
            ->where('location', $this->location)
            ->get();

        return json_decode($config->config ?? '[]', true) ?? [];
    }

    private function saveWidgetConfig(): void {
        Capsule::table('mod_myaddon_widget_config')
            ->updateOrInsert(
                ['location' => $this->location],
                ['config' => json_encode(array_values($this->widgets))]
            );
    }
}
```

### 2. Widget Database Schema

```php
<?php
// Migration for widget tables

use WHMCS\Database\Capsule;

Capsule::schema()->create('mod_myaddon_widget_config', function($t) {
    $t->string('location', 50)->primary();
    $t->longText('config');
    $t->timestamps();
});

Capsule::schema()->create('mod_myaddon_widget_cache', function($t) {
    $t->string('widget_id', 100);
    $t->longText('data');
    $t->timestamp('cached_at');

    $t->primary('widget_id');
    $t->index('cached_at');
});
```

## Checklist

- [ ] Widget extends AbstractWidget (if using new system)
- [ ] Proper cache timeout configured
- [ ] Widget title and description defined
- [ ] Configurable parameters implemented
- [ ] Responsive design for all screen sizes
- [ ] AJAX refresh functionality (if needed)
- [ ] Error handling for data fetching
- [ ] Loading states for async operations

---

**Related Skills:**
- whmcs-widget-builder
- whmcs-admin-panel-development
- whmcs-reporting
- whmcs-ajax-patterns

**Reference:**
- WHMCS Widget System: https://developers.whmcs.com/widgets/