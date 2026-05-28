# WHMCS Module Widget Reference

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This reference covers the creation of widgets for WHMCS, including admin dashboard widgets, client area widgets, and widget configuration options.

---

## Widget Structure

### Widget Directory

```
modules/widgets/
  YourWidget.php        # Main widget class
  YourWidget.tpl        # Smarty template (optional)
  settings.php          # Widget settings (optional)
```

## Admin Dashboard Widgets

### Basic Widget Class

```php
<?php
/**
 * Admin Dashboard Widget
 * 
 * Place in /modules/widgets/YourWidget.php
 */

namespace WHMCS\Module\Widget;

class YourWidget implements \WHMCS\Module\Contracts\WidgetInterface
{
    /**
     * Unique widget identifier
     */
    protected $name = 'YourWidget';
    
    /**
     * Widget title
     */
    protected $title = 'Your Widget Title';
    
    /**
     * Icon class (Font Awesome)
     */
    protected $icon = 'fa-chart-line';
    
    /**
     * Widget height in units (1 unit = 85px)
     */
    protected $height = 3;
    
    /**
     * Widget width in units
     */
    protected $width = 1;
    
    /**
     * Required permission to view widget
     */
    protected $requiredPermission = 'Your Widget Access';
    
    /**
     * Get widget data
     * 
     * @param array $params Widget parameters
     * @return array|string
     */
    public function getData(array $params)
    {
        // Fetch widget data
        $stats = WHMCS\Database\Capsule::table('mod_your_table')
            ->selectRaw('
                COUNT(*) as total,
                SUM(CASE WHEN status = "active" THEN 1 ELSE 0 END) as active,
                SUM(CASE WHEN status = "pending" THEN 1 ELSE 0 END) as pending
            ')
            ->first();
        
        return [
            'total'   => $stats->total ?? 0,
            'active'  => $stats->active ?? 0,
            'pending' => $stats->pending ?? 0,
        ];
    }
    
    /**
     * Get widget HTML
     * 
     * @param array $data Widget data from getData()
     * @param array $params Widget parameters
     * @return string
     */
    public function getHtml(array $data, array $params = [])
    {
        $smarty = \WHMCS\Engine\Smarty\Engine::getInstance();
        
        $smarty->assign('widget_total', $data['total']);
        $smarty->assign('widget_active', $data['active']);
        $smarty->assign('widget_pending', $data['pending']);
        $smarty->assign('widget_url', 'addon_module.php?module=your_mod');
        
        return $smarty->fetch('module_your_widget.tpl');
    }
    
    /**
     * Additional CSS (optional)
     */
    public function getCss()
    {
        return <<<CSS
.widget-yourwidget .stat-value {
    font-size: 24px;
    font-weight: bold;
}
CSS;
    }
    
    /**
     * Additional JavaScript (optional)
     */
    public function getJs()
    {
        return <<<JS
$(document).ready(function() {
    $('.widget-yourwidget').on('click', '.stat-value', function() {
        console.log('Widget clicked');
    });
});
JS;
    }
}
```

### Widget Template

```smarty
<div class="widget-inner">
    <div class="widget-header">
        <i class="fa fa-chart-line"></i>
        {$title}
        <div class="widget-actions">
            <a href="{$widget_url}"><i class="fa fa-cog"></i></a>
        </div>
    </div>
    
    <div class="widget-content">
        <div class="stats-container">
            <div class="stat-item">
                <span class="stat-value">{$widget_total}</span>
                <span class="stat-label">Total Records</span>
            </div>
            
            <div class="stat-item">
                <span class="stat-value text-success">{$widget_active}</span>
                <span class="stat-label">Active</span>
            </div>
            
            <div class="stat-item">
                <span class="stat-value text-warning">{$widget_pending}</span>
                <span class="stat-label">Pending</span>
            </div>
        </div>
    </div>
    
    <div class="widget-footer">
        <a href="{$widget_url}">View All <i class="fa fa-arrow-right"></i></a>
    </div>
</div>
```

### Widget Registration

```php
/**
 * Register widget hook (in addon module)
 */
function your_addon_registerHooks()
{
    return [
        'AdminAreaWidget' => 'registerYourWidget',
    ];
}

function registerYourWidget()
{
    add_hook('AdminAreaWidget', 1, function() {
        return new \WHMCS\Module\Widget\YourWidget();
    });
}
```

## Client Area Widgets

### Client Widget Class

```php
<?php
/**
 * Client Area Widget
 */

namespace WHMCS\Module\Widget;

class ClientYourWidget implements \WHMCS\Module\Contracts\ClientWidgetInterface
{
    protected $name = 'ClientYourWidget';
    protected $title = 'Your Widget';
    protected $icon = 'fa-star';
    protected $description = 'Displays important information';
    
    /**
     * Widget loading conditions
     */
    public function loadCallback(array $params)
    {
        // Only load for logged in clients
        if (!$_SESSION['uid']) {
            return false;
        }
        
        // Check if module is active for client
        $clientData = WHMCS\Database\Capsule::table('mod_your_table')
            ->where('client_id', $_SESSION['uid'])
            ->exists();
        
        return $clientData;
    }
    
    /**
     * Get widget data
     */
    public function getData(array $params)
    {
        $clientId = $_SESSION['uid'];
        
        // Fetch client-specific data
        $data = WHMCS\Database\Capsule::table('mod_your_table')
            ->where('client_id', $clientId)
            ->first();
        
        return [
            'status'       => $data->status ?? 'inactive',
            'last_updated' => $data->updated_at ?? null,
            'config'       => json_decode($data->config ?? '{}', true),
        ];
    }
    
    /**
     * Get widget HTML
     */
    public function getHtml(array $data, array $params = [])
    {
        $smarty = \WHMCS\Engine\Smarty\Engine::getInstance();
        
        $smarty->assign('status', $data['status']);
        $smarty->assign('last_updated', $data['last_updated']);
        $smarty->assign('config', $data['config']);
        
        return $smarty->fetch('client_your_widget.tpl');
    }
    
    /**
     * Widget display order
     */
    public function getDisplayOrder()
    {
        return 100;
    }
    
    /**
     * Required permissions
     */
    public function getRequiredPermission()
    {
        return null; // null for client widgets, or specific permission
    }
}
```

### Client Widget Smarty Template

```smarty
<div class="panel panel-{if $status == 'active'}success{else}default{/if}">
    <div class="panel-heading">
        <i class="fa fa-star"></i>
        <span>Your Widget</span>
    </div>
    
    <div class="panel-body">
        {if $status == 'active'}
            <div class="widget-status">
                <strong>Status:</strong> Active
            </div>
            
            <div class="widget-info">
                <strong>Last Updated:</strong>
                {$last_updated|date_format:"Y-m-d H:i"}
            </div>
            
            <div class="widget-config">
                {if $config.some_setting}
                    <span class="label label-success">Enabled</span>
                {else}
                    <span class="label label-default">Disabled</span>
                {/if}
            </div>
        {else}
            <p class="text-muted">
                This widget will be activated once you configure it.
            </p>
            <a href="clientarea.php?action=account&mod=your_addon" class="btn btn-default btn-sm">
                Configure
            </a>
        {/if}
    </div>
</div>
```

## Widget Configuration

### Setting Definitions

```php
/**
 * Widget settings
 */
function getWidgetSettings()
{
    return [
        'display_title' => [
            'FriendlyName' => 'Display Title',
            'Type'         => 'text',
            'Default'      => 'Your Widget',
            'Description'  => 'Title to display in widget header',
        ],
        'show_total' => [
            'FriendlyName' => 'Show Total',
            'Type'         => 'yesno',
            'Default'      => 'yes',
            'Description'  => 'Show total count',
        ],
        'show_pending' => [
            'FriendlyName' => 'Show Pending',
            'Type'         => 'yesno',
            'Default'      => 'yes',
            'Description'  => 'Show pending count',
        ],
        'refresh_interval' => [
            'FriendlyName' => 'Refresh Interval',
            'Type'         => 'dropdown',
            'Options'      => [
                '0'   => 'Manual Only',
                '60'  => 'Every Minute',
                '300' => 'Every 5 Minutes',
                '900' => 'Every 15 Minutes',
            ],
            'Default'      => '300',
            'Description'  => 'How often to refresh widget data',
        ],
    ];
}

/**
 * Get setting
 */
function getWidgetSetting($key, $default = null)
{
    static $settings = null;
    
    if ($settings === null) {
        $settings = get_config_var('your_widget_settings') ?? [];
    }
    
    return $settings[$key] ?? $default;
}
```

## Widget Styling

### CSS Guidelines

```css
/* Widget Standard Styling */
.widget {
    background: #fff;
    border-radius: 4px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    overflow: hidden;
}

.widget-header {
    background: #f5f5f5;
    padding: 10px 15px;
    font-weight: 600;
    border-bottom: 1px solid #ddd;
}

.widget-header i {
    margin-right: 8px;
    color: #555;
}

.widget-content {
    padding: 15px;
}

.widget-footer {
    background: #f9f9f9;
    padding: 10px 15px;
    border-top: 1px solid #ddd;
    font-size: 12px;
}

/* Stats Styling */
.stats-container {
    display: flex;
    justify-content: space-between;
}

.stat-item {
    text-align: center;
    flex: 1;
}

.stat-value {
    display: block;
    font-size: 24px;
    font-weight: bold;
    color: #333;
}

.stat-label {
    display: block;
    font-size: 11px;
    color: #777;
    text-transform: uppercase;
}

/* Color Variations */
.text-success { color: #5cb85c; }
.text-warning { color: #f0ad4e; }
.text-danger { color: #d9534f; }
.text-info { color: #5bc0de; }
```

## Widget Examples

### Activity Graph Widget

```php
class ActivityGraphWidget implements \WHMCS\Module\Contracts\WidgetInterface
{
    protected $name = 'ActivityGraph';
    protected $title = 'Activity Chart';
    protected $icon = 'fa-chart-bar';
    protected $height = 4;
    
    public function getData(array $params)
    {
        // Get last 7 days of activity
        $startDate = date('Y-m-d', strtotime('-7 days'));
        
        $activity = [];
        for ($i = 0; $i < 7; $i++) {
            $date = date('Y-m-d', strtotime("+{$i} days", strtotime($startDate)));
            
            $count = WHMCS\Database\Capsule::table('mod_your_activity')
                ->whereRaw("DATE(created_at) = ?", [$date])
                ->count();
            
            $activity[] = [
                'date'  => $date,
                'count' => $count,
                'label' => date('D', strtotime($date)),
            ];
        }
        
        return ['activity' => $activity];
    }
    
    public function getHtml(array $data, array $params = [])
    {
        $smarty = \WHMCS\Engine\Smarty\Engine::getInstance();
        $smarty->assign('activity', $data['activity']);
        
        return $smarty->fetch('activity_graph_widget.tpl');
    }
}
```

### Quick Links Widget

```php
class QuickLinksWidget implements \WHMCS\Module\Contracts\WidgetInterface
{
    protected $name = 'QuickLinks';
    protected $title = 'Quick Links';
    protected $icon = 'fa-link';
    protected $height = 2;
    
    public function getData(array $params)
    {
        $adminId = $_SESSION['adminid'];
        
        // Get user-specific quick links
        $links = WHMCS\Database\Capsule::table('mod_your_links')
            ->where('admin_id', $adminId)
            ->where('is_active', 1)
            ->orderBy('display_order')
            ->get();
        
        return ['links' => $links];
    }
    
    public function getHtml(array $data, array $params = [])
    {
        $smarty = \WHMCS\Engine\Smarty\Engine::getInstance();
        $smarty->assign('links', $data['links']);
        $smarty->assign('module_link', 'addonmodules.php?module=your_addon');
        
        return $smarty->fetch('quick_links_widget.tpl');
    }
}
```

## Widget Debugging

```php
/**
 * Widget debug output
 */
public function getData(array $params)
{
    try {
        $data = $this->fetchData($params);
        return $data;
        
    } catch (\Exception $e) {
        return [
            'error' => $e->getMessage(),
            '_debug' => [
                'trace' => $e->getTraceAsString(),
                'params' => $params,
            ],
        ];
    }
}
```

---

## Related Skills and Workflows

- `addon-module-developer-guide` - Addon development
- `smarty-template-reference` - Template development
- `client-area-theming` - Client area customization
- `admin-area-customization` - Admin area customization
