# WHMCS Widget From Scratch Workflow

## Description
Create a custom dashboard widget for WHMCS admin area.

## Prerequisites
- WHMCS 7.0+
- PHP 8.1+
- Admin access

## Steps

### Step 1: Create Widget Directory
```bash
mkdir -p /var/www/whmcs/modules/widgets/clicodes_widget
```

### Step 2: Create Widget File
```php
<?php
/**
 * WHMCS Admin Dashboard Widget - CLICodes Stats
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Widget meta data
 */
function clicodes_widget_MetaData()
{
    return [
        'name' => 'CLICodes Statistics',
        'description' => 'Display key business statistics',
        'version' => '1.0.0',
        'author' => 'Your Company',
        'defaultsize' => 'col-md-3 col-lg-2',
    ];
}

/**
 * Get widget settings
 */
function clicodes_widget_config()
{
    return [
        'displayCount' => [
            'Type' => 'dropdown',
            'Label' => 'Stats to Display',
            'Options' => [
                'all' => 'All Statistics',
                'revenue' => 'Revenue Only',
                'clients' => 'Clients Only',
                'services' => 'Services Only',
            ],
            'Default' => 'all',
        ],
        'refreshInterval' => [
            'Type' => 'text',
            'Label' => 'Refresh Interval (seconds)',
            'Default' => '300',
        ],
        'showCharts' => [
            'Type' => 'yesno',
            'Label' => 'Show Charts',
            'Default' => 'on',
        ],
    ];
}

/**
 * Output widget content
 */
function clicodes_widget_output($params)
{
    $config = $params['widgetConfig'];
    $displayCount = $config['displayCount'] ?? 'all';
    
    // Get statistics
    $stats = clicodes_widget_getStats();
    
    $html = '<div class="widget-content">';
    
    // Revenue Stats
    if ($displayCount === 'all' || $displayCount === 'revenue') {
        $html .= clicodes_widget_renderStat(
            'Monthly Revenue',
            formatCurrency($stats['monthly_revenue']),
            'fa-dollar-sign',
            'success'
        );
    }
    
    // Client Stats
    if ($displayCount === 'all' || $displayCount === 'clients') {
        $html .= clicodes_widget_renderStat(
            'Active Clients',
            number_format($stats['active_clients']),
            'fa-users',
            'info'
        );
    }
    
    // Services Stats
    if ($displayCount === 'all' || $displayCount === 'services') {
        $html .= clicodes_widget_renderStat(
            'Active Services',
            number_format($stats['active_services']),
            'fa-server',
            'primary'
        );
    }
    
    // Pending Tickets
    $html .= clicodes_widget_renderStat(
        'Pending Tickets',
        number_format($stats['pending_tickets']),
        'fa-ticket-alt',
        'warning'
    );
    
    $html .= '</div>';
    
    // Add refresh functionality
    $refreshInterval = $config['refreshInterval'] ?? 300;
    $html .= '<script>
        setInterval(function() {
            location.reload();
        }, ' . ($refreshInterval * 1000) . ');
    </script>';
    
    return $html;
}

/**
 * Render individual stat
 */
function clicodes_widget_renderStat($title, $value, $icon, $color)
{
    $colorClass = [
        'success' => 'bg-success',
        'info' => 'bg-info',
        'primary' => 'bg-primary',
        'warning' => 'bg-warning',
    ][$color] ?? 'bg-secondary';
    
    return <<<HTML
<div class="stat-box">
    <div class="stat-icon {$colorClass}">
        <i class="fas {$icon}"></i>
    </div>
    <div class="stat-content">
        <h4>{$value}</h4>
        <span class="stat-title">{$title}</span>
    </div>
</div>
HTML;
}

/**
 * Get statistics from database
 */
function clicodes_widget_getStats()
{
    $pdo = Capsule::connection()->getPdo();
    
    // Monthly revenue
    $stmt = $pdo->prepare("
        SELECT SUM(total) as revenue 
        FROM tblinvoices 
        WHERE status = 'Paid' 
        AND date >= DATE_FORMAT(NOW(), '%Y-%m-01')
    ");
    $stmt->execute();
    $monthlyRevenue = $stmt->fetch(PDO::FETCH_ASSOC)['revenue'] ?? 0;
    
    // Active clients
    $stmt = $pdo->prepare("
        SELECT COUNT(*) as count 
        FROM tblclients 
        WHERE status = 'Active'
    ");
    $stmt->execute();
    $activeClients = $stmt->fetch(PDO::FETCH_ASSOC)['count'] ?? 0;
    
    // Active services
    $stmt = $pdo->prepare("
        SELECT COUNT(*) as count 
        FROM tblhosting 
        WHERE domainstatus = 'Active'
    ");
    $stmt->execute();
    $activeServices = $stmt->fetch(PDO::FETCH_ASSOC)['count'] ?? 0;
    
    // Pending tickets
    $stmt = $pdo->prepare("
        SELECT COUNT(*) as count 
        FROM tbltickets 
        WHERE status IN ('Open', 'Awaiting Reply')
    ");
    $stmt->execute();
    $pendingTickets = $stmt->fetch(PDO::FETCH_ASSOC)['count'] ?? 0;
    
    return [
        'monthly_revenue' => $monthlyRevenue,
        'active_clients' => $activeClients,
        'active_services' => $activeServices,
        'pending_tickets' => $pendingTickets,
    ];
}
```

### Step 3: Install Widget
```bash
# Copy widget
cp -r clicodes_widget /var/www/whmcs/modules/widgets/

# Set permissions
chown -R www-data:www-data /var/www/whmcs/modules/widgets/clicodes_widget
chmod 644 /var/www/whmcs/modules/widgets/clicodes_widget/*.php

# Enable in WHMCS Admin
# Go to: Configuration > System Settings > Administrators
# Click on admin > Widget Preferences
# Enable CLICodes Statistics widget
```

## Tags
- widget
- dashboard
- admin
- module-development