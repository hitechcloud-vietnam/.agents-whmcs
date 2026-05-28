# WHMCS Widget API Reference

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-widget-builder`, `admin-area-customization`

---

## Overview

Widgets are dashboard components that display information in the WHMCS admin and client areas. They provide at-a-glance views of important data and can be placed anywhere on dashboard pages.

---

## Widget Structure

### File Location

```
modules/widgets/
├── AdminWidget.php    # Admin dashboard widget
├── ClientWidget.php   # Client area widget
└── templates/
    ├── admin_widget.tpl
    └── client_widget.tpl
```

---

## Admin Dashboard Widgets

### Widget Interface

```php
<?php
namespace WHMCS\Module\Contracts;

interface WidgetInterface
{
    /**
     * Get unique widget name
     */
    public function getName(): string;

    /**
     * Get widget title
     */
    public function getTitle(): string;

    /**
     * Get widget icon (Font Awesome class)
     */
    public function getIcon(): string;

    /**
     * Get widget height in units (1 unit = 85px)
     */
    public function getHeight(): int;

    /**
     * Fetch widget data
     */
    public function getData(array $params = []): mixed;

    /**
     * Render widget HTML
     */
    public function getHtml(array $data, array $params = []): string;
}
```

### Basic Admin Widget

```php
<?php
/**
 * Admin Dashboard Widget
 */

namespace WHMCS\Module\Widget;

class StatsWidget implements \WHMCS\Module\Contracts\WidgetInterface
{
    protected string $name = 'StatsWidget';
    protected string $title = 'Quick Stats';
    protected string $icon = 'fa-chart-line';
    protected int $height = 3;
    protected int $width = 1;
    protected ?string $requiredPermission = null;

    /**
     * Get widget name
     */
    public function getName(): string
    {
        return $this->name;
    }

    /**
     * Get widget title
     */
    public function getTitle(): string
    {
        return $this->title;
    }

    /**
     * Get widget icon
     */
    public function getIcon(): string
    {
        return $this->icon;
    }

    /**
     * Get widget height
     */
    public function getHeight(): int
    {
        return $this->height;
    }

    /**
     * Get widget width
     */
    public function getWidth(): int
    {
        return $this->width;
    }

    /**
     * Get required permission
     */
    public function getRequiredPermission(): ?string
    {
        return $this->requiredPermission;
    }

    /**
     * Fetch widget data
     */
    public function getData(array $params = []): array
    {
        $stats = [];

        // Today's stats
        $stats['today_revenue'] = Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereRaw("DATE(datepaid) = CURDATE()")
            ->sum('total');

        $stats['today_orders'] = Capsule::table('tblorders')
            ->whereRaw("DATE(date) = CURDATE()")
            ->count();

        $stats['new_tickets'] = Capsule::table('tbltickets')
            ->whereRaw("DATE(created) = CURDATE()")
            ->count();

        $stats['active_services'] = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count();

        return $stats;
    }

    /**
     * Render widget HTML
     */
    function getHtml(array $data, array $params = []): string
    {
        $smarty = \WHMCS\Engine\Smarty\Engine::getInstance();

        $smarty->assign('today_revenue', formatCurrency($data['today_revenue'] ?? 0));
        $smarty->assign('today_orders', $data['today_orders'] ?? 0);
        $smarty->assign('new_tickets', $data['new_tickets'] ?? 0);
        $smarty->assign('active_services', $data['active_services'] ?? 0);

        return $smarty->fetch('module_stats_widget.tpl');
    }

    /**
     * Additional CSS (optional)
     */
    public function getCss(): string
    {
        return <<<CSS
.widget-stats-widget .stat-card {
    background: #f8f9fa;
    border-radius: 8px;
    padding: 15px;
    margin-bottom: 10px;
}
.widget-stats-widget .stat-value {
    font-size: 24px;
    font-weight: bold;
    color: #333;
}
.widget-stats-widget .stat-label {
    font-size: 12px;
    color: #666;
    text-transform: uppercase;
}
CSS;
    }

    /**
     * JavaScript (optional)
     */
    public function getJs(): string
    {
        return <<<JS
$(document).ready(function() {
    $('.widget-stats-widget').on('click', '.stat-card', function() {
        var type = $(this).data('type');
        window.location.href = 'index.php?widget=' + type;
    });
});
JS;
    }
}
```

---

## Widget Template

### Admin Widget Template

```smarty
<div class="widget widget-stats-widget" id="widget-{$widget.id}">
    <div class="widget-header">
        <i class="fa {$widget.icon}"></i>
        {$widget.title}
        <div class="widget-actions">
            <a href="#" class="refresh-widget"><i class="fa fa-refresh"></i></a>
            <a href="#" class="minimize-widget"><i class="fa fa-minus"></i></a>
        </div>
    </div>

    <div class="widget-content">
        <div class="row">
            <div class="col-md-3">
                <div class="stat-card" data-type="revenue">
                    <div class="stat-value">{$today_revenue}</div>
                    <div class="stat-label">Today's Revenue</div>
                </div>
            </div>
            <div class="col-md-3">
                <div class="stat-card" data-type="orders">
                    <div class="stat-value">{$today_orders}</div>
                    <div class="stat-label">New Orders</div>
                </div>
            </div>
            <div class="col-md-3">
                <div class="stat-card" data-type="tickets">
                    <div class="stat-value">{$new_tickets}</div>
                    <div class="stat-label">Support Tickets</div>
                </div>
            </div>
            <div class="col-md-3">
                <div class="stat-card" data-type="services">
                    <div class="stat-value">{$active_services}</div>
                    <div class="stat-label">Active Services</div>
                </div>
            </div>
        </div>
    </div>

    <div class="widget-footer">
        <a href="index.php">View Dashboard <i class="fa fa-arrow-right"></i></a>
    </div>
</div>
```

---

## Client Area Widgets

### Client Widget Interface

```php
<?php
namespace WHMCS\Module\Contracts;

interface ClientWidgetInterface
{
    /**
     * Get unique widget name
     */
    public function getName(): string;

    /**
     * Get display title
     */
    public function getTitle(): string;

    /**
     * Get widget icon
     */
    public function getIcon(): string;

    /**
     * Check if widget should be loaded
     */
    public function loadCallback(array $params): bool;

    /**
     * Fetch widget data
     */
    public function getData(array $params): array;

    /**
     * Render widget HTML
     */
    public function getHtml(array $data, array $params = []): string;

    /**
     * Get display order
     */
    public function getDisplayOrder(): int;
}
```

### Client Service Widget

```php
<?php
/**
 * Client Area Service Widget
 */

namespace WHMCS\Module\Widget;

class ServiceStatusWidget implements \WHMCS\Module\Contracts\ClientWidgetInterface
{
    protected string $name = 'ServiceStatus';
    protected string $title = 'Service Status';
    protected string $icon = 'fa-server';
    protected int $displayOrder = 100;

    /**
     * Check if user has services
     */
    public function loadCallback(array $params): bool
    {
        if (!$_SESSION['uid']) {
            return false;
        }

        return Capsule::table('tblhosting')
            ->where('userid', $_SESSION['uid'])
            ->exists();
    }

    /**
     * Fetch user services data
     */
    public function getData(array $params): array
    {
        $userId = $_SESSION['uid'];

        $services = Capsule::table('tblhosting as h')
            ->join('tblproducts as p', 'p.id', '=', 'h.packageid')
            ->where('h.userid', $userId)
            ->where('h.domainstatus', 'Active')
            ->select('h.id', 'h.domain', 'h.dedicatedip', 'p.name as product_name', 'h.nextduedate')
            ->orderBy('h.nextduedate', 'asc')
            ->limit(5)
            ->get();

        return [
            'services' => $services,
            'total_services' => Capsule::table('tblhosting')
                ->where('userid', $userId)
                ->where('domainstatus', 'Active')
                ->count(),
        ];
    }

    /**
     * Render widget HTML
     */
    public function getHtml(array $data, array $params = []): string
    {
        $smarty = \WHMCS\Engine\Smarty\Engine::getInstance();

        $smarty->assign('services', $data['services']);
        $smarty->assign('total_services', $data['total_services']);

        return $smarty->fetch('modules/widgets/client_service_status.tpl');
    }

    /**
     * Widget display order
     */
    public function getDisplayOrder(): int
    {
        return $this->displayOrder;
    }
}
```

### Client Widget Template

```smarty
<div class="panel panel-default">
    <div class="panel-heading">
        <i class="fa {$widget.icon}"></i>
        <span>{$widget.title}</span>
    </div>

    <div class="panel-body">
        {if $services}
            <div class="service-list">
                {foreach $services as $service}
                    <div class="service-item">
                        <div class="service-info">
                            <strong>{$service->product_name}</strong>
                            <br>
                            <small class="text-muted">{$service->domain}</small>
                        </div>
                        <div class="service-meta">
                            <span class="label label-success">Active</span>
                            <br>
                            <small>Next due: {$service->nextduedate|date_format}</small>
                        </div>
                    </div>
                {/foreach}
            </div>
        {else}
            <p class="text-muted">No active services found.</p>
        {/if}
    </div>

    {if $total_services > 5}
        <div class="panel-footer">
            <a href="clientarea.php?action=services">View All ({$total_services})</a>
        </div>
    {/if}
</div>
```

---

## Widget Data Fetching

### Database Queries

```php
/**
 * Complex data aggregation
 */
public function getData(array $params = []): array
{
    // Monthly revenue chart data
    $monthlyRevenue = Capsule::table('tblinvoices')
        ->selectRaw("DATE_FORMAT(datepaid, '%Y-%m') as month, SUM(total) as revenue")
        ->where('status', 'Paid')
        ->whereRaw("datepaid >= DATE_SUB(CURDATE(), INTERVAL 6 MONTH)")
        ->groupByRaw("DATE_FORMAT(datepaid, '%Y-%m')")
        ->orderByRaw("DATE_FORMAT(datepaid, '%Y-%m')")
        ->get();

    // Service distribution
    $serviceDistribution = Capsule::table('tblhosting as h')
        ->join('tblproducts as p', 'p.id', '=', 'h.packageid')
        ->join('tblproduct_groups as g', 'g.id', '=', 'p.gid')
        ->selectRaw("g.name as group_name, COUNT(*) as count")
        ->where('h.domainstatus', 'Active')
        ->groupBy('g.id')
        ->get();

    // Recent activity
    $recentActivity = Capsule::table('tblactivitylog')
        ->selectRaw("DATE_FORMAT(date, '%Y-%m-%d %H:%i') as time, CONCAT('Client #', userid, ' - ', description) as activity")
        ->orderBy('date', 'desc')
        ->limit(10)
        ->get();

    return [
        'revenue' => $monthlyRevenue,
        'distribution' => $serviceDistribution,
        'activity' => $recentActivity,
    ];
}
```

---

## Widget Settings

### Widget Configuration

```php
<?php
/**
 * Widget settings management
 */
class WidgetSettings {
    private int $adminId;
    private string $widgetName;
    private array $cache = [];

    public function __construct(int $adminId, string $widgetName)
    {
        $this->adminId = $adminId;
        $this->widgetName = $widgetName;
    }

    public function get(string $key, $default = null) {
        $settings = $this->getAll();
        return $settings[$key] ?? $default;
    }

    public function set(string $key, $value): void {
        Capsule::table('mod_widget_settings')
            ->updateOrInsert(
                ['admin_id' => $this->adminId, 'widget' => $this->widgetName, 'setting_key' => $key],
                ['setting_value' => is_array($value) ? json_encode($value) : $value]
            );
        $this->cache[$key] = $value;
    }

    private function getAll(): array {
        if (!empty($this->cache)) {
            return $this->cache;
        }

        $rows = Capsule::table('mod_widget_settings')
            ->where('admin_id', $this->adminId)
            ->where('widget', $this->widgetName)
            ->get();

        foreach ($rows as $row) {
            $this->cache[$row->setting_key] = $row->setting_value;
        }

        return $this->cache;
    }
}
```

### Widget Settings UI

```php
/**
 * Widget with settings
 */
class ConfigurableWidget implements \WHMCS\Module\Contracts\WidgetInterface {
    // ...

    public function getData(array $params = []): array {
        $settings = new WidgetSettings($_SESSION['adminid'], $this->name);

        $refreshInterval = $settings->get('refresh_interval', 300);
        $showNumbers = $settings->get('show_numbers', true);

        $data = $this->fetchData();

        return [
            'data' => $data,
            'show_numbers' => $showNumbers,
            'refresh_interval' => $refreshInterval,
        ];
    }

    public function getSettings(): array {
        return [
            'refresh_interval' => [
                'FriendlyName' => 'Refresh Interval',
                'Type' => 'dropdown',
                'Options' => [
                    '0' => 'Manual Only',
                    '60' => 'Every Minute',
                    '300' => 'Every 5 Minutes',
                    '900' => 'Every 15 Minutes',
                ],
                'Default' => '300',
            ],
            'show_numbers' => [
                'FriendlyName' => 'Show Numbers',
                'Type' => 'yesno',
                'Default' => 'yes',
            ],
        ];
    }
}
```

---

## Widget Hook Registration

### Registering Widgets

```php
<?php
/**
 * Register admin widgets via hook
 */
function mymodule_register_widgets() {
    add_hook('AdminAreaWidget', 1, function() {
        return new \WHMCS\Module\Widget\StatsWidget();
    });
}

/**
 * Register with widget factory
 */
function mymodule_register_widgets_v2() {
    add_hook('AdminAreaWidget', 1, function() {
        return [
            'name' => 'StatsWidget',
            'class' => \WHMCS\Module\Widget\StatsWidget::class,
            'title' => 'Quick Stats',
            'icon' => 'fa-chart-line',
            'description' => 'Display key statistics',
            'position' => 'sidebar-right',
        ];
    });
}
```

---

## Client Widget Registration

```php
<?php
/**
 * Register client widgets
 */
function mymodule_register_client_widgets() {
    add_hook('ClientAreaWidget', 1, function() {
        $widget = new \WHMCS\Module\Widget\ServiceStatusWidget();

        if ($widget->loadCallback([])) {
            return $widget;
        }

        return null;
    });
}
```

---

## Widget Styling

### CSS Guidelines

```css
/* Base widget styling */
.widget {
    background: #fff;
    border-radius: 4px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    margin-bottom: 20px;
}

.widget-header {
    background: #f5f5f5;
    padding: 12px 15px;
    border-bottom: 1px solid #e5e5e5;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.widget-header i { margin-right: 8px; color: #555; }
.widget-header .widget-title { font-weight: 600; }

.widget-content { padding: 15px; }
.widget-footer {
    background: #f9f9f9;
    padding: 10px 15px;
    border-top: 1px solid #e5e5e5;
    font-size: 12px;
}

/* Responsive grid */
@media (max-width: 768px) {
    .widget { margin-bottom: 15px; }
    .widget-content { padding: 10px; }
}
```

---

## Widget Debugging

```php
<?php
public function getData(array $params = []): array
{
    try {
        $data = $this->fetchData($params);
        return $data;

    } catch (\Exception $e) {
        logActivity('Widget error: ' . $e->getMessage());

        return [
            'error' => true,
            'message' => 'Failed to load widget data',
            'details' => $_SESSION['adminid'] ?? 'unknown',
        ];
    }
}

public function getHtml(array $data, array $params = []): string
{
    if (!empty($data['error'])) {
        return '<div class="widget widget-error">
            <div class="widget-content">
                <p class="text-danger">' . $data['message'] . '</p>
            </div>
        </div>';
    }

    // Normal rendering
    // ...
}
```

---

## Best Practices

1. **Keep widgets simple** - Display summary data only
2. **Implement caching** - Cache data to reduce DB queries
3. **Handle errors gracefully** - Show fallback content on errors
4. **Responsive design** - Works on all screen sizes
5. **Lazy loading** - Load data on-demand when possible
6. **Refresh mechanism** - Provide manual refresh option
7. **Permission checks** - Verify admin permissions
8. **Performance** - Keep queries optimized

---

## Related Documentation

- [Module Widget Reference](module-widget-reference.md)
- [Widget Builder Skill](../skills/whmcs-widget-builder)
- [Admin Area Customization](admin-area-customization.md)
