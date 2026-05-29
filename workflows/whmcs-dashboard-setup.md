# WHMCS Dashboard Setup Workflow

## Overview
Configure the WHMCS dashboard with widgets, metrics, and custom layouts.

## Prerequisites
- WHMCS v8.0+
- Admin dashboard access

## Step-by-Step Guide

### Step 1: Admin Dashboard Widget
```php
<?php
add_hook('AdminHomeWidgets', 1, function() {
    return new class extends \WHMCS\Module\AbstractWidget {
        public $title = 'Monthly Revenue';
        public $description = 'Shows monthly revenue stats';
        
        public function getBody(): string
        {
            $revenue = \WHMCS\Database\Capsule::table('tblaccounts')
                ->where('date', '>=', date('Y-m-01'))
                ->sum('amount');
            
            return '<div class="revenue-widget">
                <h2>$' . number_format($revenue, 2) . '</h2>
                <p>Revenue this month</p>
            </div>';
        }
        
        public function getData(): array
        {
            return [
                'monthly_revenue' => $this->getMonthlyRevenue(),
                'active_clients' => $this->getActiveClients(),
            ];
        }
    };
});
```

### Step 2: Widget Positioning
```php
<?php
add_hook('AdminHomeWidgetSortable', 1, function() {
    return [
        'order' => [
            'MonthlyRevenue' => 1,
            'SupportTickets' => 2,
            'RecentOrders' => 3,
        ],
    ];
});
```

## Checklist
- Dashboard widgets added
- Metrics configured
- Layout customized
- Performance optimized
