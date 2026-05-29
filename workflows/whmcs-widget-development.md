# WHMCS Widget Development Workflow

## Purpose
Create custom dashboard widgets for WHMCS admin area

## Prerequisites
- WHMCS installed
- PHP development knowledge
- Admin access

## Step 1: Create Widget Directory

```bash
mkdir -p /var/www/whmcs/modules/widgets/MyCustomWidget
```

## Step 2: Create Widget Class

```php
<?php
// MyCustomWidget.php

namespace WHMCS\Module\Widget;

use WHMCS\Authentication\CurrentUser;
use WHMCS\Carbon;
use WHMCS\Module\Widget\WidgetInterface;

class MyCustomWidget implements WidgetInterface
{
    protected $data = [];
    
    public function getId()
    {
        return 'my_custom_widget';
    }
    
    public function getName()
    {
        return 'Custom Statistics';
    }
    
    public function getDescription()
    {
        return 'Displays custom statistics and metrics';
    }
    
    public function getVariables()
    {
        return [];
    }
    
    public function getUserVariables()
    {
        return [];
    }
    
    public function renderOutput(array $params)
    {
        $stats = $this->getStatistics();
        
        return <<<HTML
<div class="widget-content">
    <div class="stats-grid">
        <div class="stat-box">
            <h4>Total Clients</h4>
            <p class="stat-value">{$stats['totalClients']}</p>
        </div>
        <div class="stat-box">
            <h4>Active Services</h4>
            <p class="stat-value">{$stats['activeServices']}</p>
        </div>
        <div class="stat-box">
            <h4>Monthly Revenue</h4>
            <p class="stat-value">\${$stats['monthlyRevenue']}</p>
        </div>
        <div class="stat-box">
            <h4>Open Tickets</h4>
            <p class="stat-value">{$stats['openTickets']}</p>
        </div>
    </div>
</div>
HTML;
    }
    
    private function getStatistics()
    {
        return [
            'totalClients' => $this->getTotalClients(),
            'activeServices' => $this->getActiveServices(),
            'monthlyRevenue' => $this->getMonthlyRevenue(),
            'openTickets' => $this->getOpenTickets(),
        ];
    }
    
    private function getTotalClients()
    {
        return \WHMCS\Database\Capsule::table('tblclients')->count();
    }
    
    private function getActiveServices()
    {
        return \WHMCS\Database\Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count();
    }
    
    private function getMonthlyRevenue()
    {
        $month = Carbon::now()->month;
        $year = Carbon::now()->year;
        
        $result = \WHMCS\Database\Capsule::table('tblinvoices')
            ->whereYear('date', $year)
            ->whereMonth('date', $month)
            ->where('status', 'Paid')
            ->sum('total');
        
        return number_format($result, 2);
    }
    
    private function getOpenTickets()
    {
        return \WHMCS\Database\Capsule::table('tbltickets')
            ->whereIn('status', ['Open', 'Awaiting Reply', 'In Progress'])
            ->count();
    }
    
    public function getAdminTtl()
    {
        return 300; // Cache for 5 minutes
    }
}
```

## Step 3: Register Widget

Navigate to: Utilities > System > Widgets

Widget should appear automatically if class is properly named and located.

## Step 4: Create Widget with Settings

```php
<?php
// AdvancedWidget.php

namespace WHMCS\Module\Widget;

use WHMCS\Module\Widget\WidgetInterface;

class AdvancedWidget implements WidgetInterface
{
    public function getId()
    {
        return 'advanced_widget';
    }
    
    public function getName()
    {
        return 'Advanced Widget';
    }
    
    public function getDescription()
    {
        return 'Widget with configurable settings';
    }
    
    public function getVariables()
    {
        return [
            'displayName' => 'Advanced Widget',
            'description' => 'Customizable widget',
            'type' => 'dynamic',
        ];
    }
    
    public function getUserVariables()
    {
        return [
            'showRevenue' => [
                'type' => 'yesno',
                'default' => '1',
                'label' => 'Show Revenue',
            ],
            'chartType' => [
                'type' => 'dropdown',
                'options' => [
                    'bar' => 'Bar Chart',
                    'line' => 'Line Chart',
                    'pie' => 'Pie Chart',
                ],
                'default' => 'bar',
                'label' => 'Chart Type',
            ],
        ];
    }
    
    public function renderOutput(array $params)
    {
        $showRevenue = $params['userVariables']['showRevenue'] ?? true;
        $chartType = $params['userVariables']['chartType'] ?? 'bar';
        
        // Generate output based on settings
        $output = '<div class="advanced-widget">';
        
        if ($showRevenue) {
            $output .= '<div class="revenue-section">' . $this->getRevenueChart($chartType) . '</div>';
        }
        
        $output .= '</div>';
        
        return $output;
    }
    
    private function getRevenueChart($type)
    {
        return '<div class="chart-placeholder">Revenue Chart (' . $type . ')</div>';
    }
}
```

## Step 5: Add Widget Assets

```bash
mkdir -p /var/www/whmcs/modules/widgets/MyCustomWidget/assets
```

### CSS
```css
/* assets/widget.css */

.widget-content {
    padding: 15px;
}

.stats-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 15px;
}

.stat-box {
    background: #f8f9fa;
    padding: 15px;
    border-radius: 4px;
    text-align: center;
}

.stat-value {
    font-size: 24px;
    font-weight: bold;
    color: #0073aa;
    margin: 5px 0 0;
}
```

## Step 6: Use Hooks for Widgets

```php
// hooks.php

add_hook('AdminHomeWidgets', 1, function($vars) {
    return [
        [
            'name' => 'Custom Stats Widget',
            'filename' => 'custom_stats',
            'title' => 'Custom Statistics',
        ],
    ];
});
```

## Widget Development Checklist

- [ ] Widget directory created
- [ ] Widget class created
- [ ] Widget registered
- [ ] Settings configured
- [ ] Assets added
- [ ] Widget tested
