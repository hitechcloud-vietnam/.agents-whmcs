# WHMCS Widget Development

## Overview

Widgets provide custom dashboard widgets in the WHMCS admin area with real-time data visualization.

## Widget Structure

```
modules/widgets/YourWidget.php
```

## Widget Implementation

```php
<?php
// modules/widgets/YourWidget.php

namespace WHMCS\Module\Widget;

class YourWidget implements WidgetInterface
{
    protected $title = 'Your Widget';
    protected $description = 'Custom widget description';
    
    public static function getId(): string
    {
        return 'your_widget';
    }
    
    public function getTitle(): string
    {
        return $this->title;
    }
    
    public function getDescription(): string
    {
        return $this->description;
    }
    
    public function getSize(): string
    {
        return 'quarter'; // quarter, third, half, full
    }
    
    public function getRequiredPermission(): ?string
    {
        return 'Dashboard Statistics'; // Optional permission requirement
    }
    
    public function getData(): array
    {
        // Fetch and return widget data
        return [
            'total_clients' => $this->getTotalClients(),
            'new_clients_today' => $this->getNewClientsToday(),
            'active_services' => $this->getActiveServices(),
            'pending_orders' => $this->getPendingOrders(),
            'open_tickets' => $this->getOpenTickets(),
        ];
    }
    
    public function generateOutput(array $data): string
    {
        $smarty = new \WHMCS\Smarty;
        
        $smarty->assign('data', $data);
        
        return $smarty->fetch('modules/widgets/yourwidget.tpl');
    }
    
    public function getRefreshInterval(): int
    {
        return 300; // 5 minutes
    }
    
    private function getTotalClients(): int
    {
        return Capsule::table('tblclients')->count();
    }
    
    private function getNewClientsToday(): int
    {
        return Capsule::table('tblclients')
            ->whereDate('created_at', date('Y-m-d'))
            ->count();
    }
    
    private function getActiveServices(): int
    {
        return Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count();
    }
    
    private function getPendingOrders(): int
    {
        return Capsule::table('tblorders')
            ->where('status', 'Pending')
            ->count();
    }
    
    private function getOpenTickets(): int
    {
        return Capsule::table('tbtickets')
            ->whereIn('status', ['Open', 'Customer Reply'])
            ->count();
    }
}
```

## Widget Template

```smarty
<div class="widget-stats">
    <div class="stat-box">
        <div class="stat-value">{$data.total_clients}</div>
        <div class="stat-label">Total Clients</div>
    </div>
    
    <div class="stat-box highlight">
        <div class="stat-value">{$data.new_clients_today}</div>
        <div class="stat-label">New Today</div>
    </div>
    
    <div class="stat-box">
        <div class="stat-value">{$data.active_services}</div>
        <div class="stat-label">Active Services</div>
    </div>
    
    <div class="stat-box">
        <div class="stat-value">{$data.pending_orders}</div>
        <div class="stat-label">Pending Orders</div>
    </div>
    
    <div class="stat-box">
        <div class="stat-value">{$data.open_tickets}</div>
        <div class="stat-label">Open Tickets</div>
    </div>
</div>

<style>
.widget-stats {
    display: grid;
    grid-template-columns: repeat(5, 1fr);
    gap: 10px;
}

.stat-box {
    text-align: center;
    padding: 15px 10px;
    background: #f8f9fa;
    border-radius: 4px;
}

.stat-box.highlight {
    background: #e3f2fd;
}

.stat-value {
    font-size: 24px;
    font-weight: bold;
    color: #333;
}

.stat-label {
    font-size: 11px;
    color: #666;
    text-transform: uppercase;
    margin-top: 5px;
}
</style>
```

## Chart Widget

```php
<?php
class SalesChartWidget implements WidgetInterface
{
    public static function getId(): string
    {
        return 'sales_chart';
    }
    
    public function getTitle(): string
    {
        return 'Sales Chart';
    }
    
    public function getSize(): string
    {
        return 'half';
    }
    
    public function getData(): array
    {
        return [
            'labels' => $this->getMonthLabels(),
            'data' => $this->getMonthlySales(),
        ];
    }
    
    public function generateOutput(array $data): string
    {
        $chartId = 'sales-chart-' . uniqid();
        
        $html = '<div class="chart-container">';
        $html .= '<canvas id="' . $chartId . '"></canvas>';
        $html .= '</div>';
        $html .= '<script>';
        $html .= 'new Chart(document.getElementById("' . $chartId . '"), {';
        $html .= 'type: "line",';
        $html .= 'data: {';
        $html .= 'labels: ' . json_encode($data['labels']) . ',';
        $html .= 'datasets: [{';
        $html .= 'label: "Monthly Sales",';
        $html .= 'data: ' . json_encode($data['data']) . ',';
        $html .= 'borderColor: "#3498db",';
        $html .= 'fill: false,';
        $html .= '}]';
        $html .= '},';
        $html .= 'options: { responsive: true, maintainAspectRatio: false }';
        $html .= '});';
        $html .= '</script>';
        
        return $html;
    }
    
    private function getMonthLabels(): array
    {
        $labels = [];
        for ($i = 5; $i >= 0; $i--) {
            $labels[] = date('M Y', strtotime("-$i months"));
        }
        return $labels;
    }
    
    private function getMonthlySales(): array
    {
        $sales = [];
        for ($i = 5; $i >= 0; $i--) {
            $month = date('Y-m', strtotime("-$i months"));
            $amount = Capsule::table('tblinvoices')
                ->where('status', 'Paid')
                ->whereRaw("DATE_FORMAT(datepaid, '%Y-%m') = ?", [$month])
                ->sum('total');
            $sales[] = (float) $amount;
        }
        return $sales;
    }
}
```

## Best Practices

1. **Implement caching** - Cache expensive queries
2. **Use appropriate sizes** - Match widget to data complexity
3. **Handle empty states** - Show helpful messages
4. **Refresh appropriately** - Set reasonable refresh intervals
5. **Check permissions** - Use getRequiredPermission for security

## Related Documentation

- [WHMCS Admin Views](/docs/whmcs-admin-views.md)
- [WHMCS Charting](/docs/whmcs-charting.md)