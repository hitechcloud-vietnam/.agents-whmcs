# WHMCS Widget Module Builder

## Concept

Widget modules add dashboard widgets to the WHMCS admin area. They display real-time data, statistics, and quick actions directly on the admin dashboard.

## File Structure

```
/modules/widgets/
├── yourwidget.php    # Widget module
└── icon.png          # Widget icon (optional)
```

## Core Widget Structure

```php
<?php
/**
 * Widget Module: Your Widget
 * Version: 1.0.0
 * Description: Dashboard widget for displaying...
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function yourwidget_meta()
{
    return [
        'name' => 'Your Widget',
        'description' => 'Display custom information',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'refresh_interval' => [
                'FriendlyName' => 'Refresh Interval',
                'Type' => 'dropdown',
                'Options' => [
                    '30' => '30 seconds',
                    '60' => '1 minute',
                    '300' => '5 minutes',
                    '600' => '10 minutes',
                ],
                'Default' => '60',
            ],
            'show_graphs' => [
                'FriendlyName' => 'Show Graphs',
                'Type' => 'yesno',
                'Default' => '1',
            ],
        ],
    ];
}

function yourwidget_output($vars)
{
    // Get widget configuration
    $refreshInterval = $vars['config']['refresh_interval'] ?? 60;
    $showGraphs = $vars['config']['show_graphs'] ?? true;
    
    // Fetch data
    $data = yourwidget_fetch_data();
    
    // Get user theme
    $theme = \WHMCS\Application\Support\Facades\Lang::getSystemTheme();
    
    // Render based on theme
    if ($theme === 'dark') {
        $bgColor = '#1a1a2e';
        $textColor = '#eaeaea';
    } else {
        $bgColor = '#ffffff';
        $textColor = '#333333';
    }
    ?>
    <div class="widget-content" 
         data-refresh="<?php echo $refreshInterval; ?>"
         data-widget="yourwidget">
        
        <div class="widget-stats">
            <div class="stat-box">
                <div class="stat-value"><?php echo number_format($data['total']); ?></div>
                <div class="stat-label">Total</div>
            </div>
            <div class="stat-box">
                <div class="stat-value" style="color: #28a745;">
                    <?php echo number_format($data['active']); ?>
                </div>
                <div class="stat-label">Active</div>
            </div>
            <div class="stat-box">
                <div class="stat-value" style="color: #dc3545;">
                    <?php echo number_format($data['pending']); ?>
                </div>
                <div class="stat-label">Pending</div>
            </div>
        </div>
        
        <?php if ($showGraphs): ?>
        <div class="widget-chart">
            <canvas id="yourwidget-chart" height="100"></canvas>
        </div>
        <?php endif; ?>
        
        <div class="widget-actions">
            <a href="yourwidget.php?action=list" class="btn btn-sm btn-default">
                View All
            </a>
            <button type="button" class="btn btn-sm btn-default" onclick="refreshYourWidget()">
                <i class="fa fa-refresh"></i>
            </button>
        </div>
    </div>
    
    <style>
        .widget-content { padding: 15px; }
        .widget-stats { display: flex; justify-content: space-between; margin-bottom: 15px; }
        .stat-box { text-align: center; flex: 1; }
        .stat-value { font-size: 24px; font-weight: bold; }
        .stat-label { font-size: 11px; color: #666; text-transform: uppercase; }
        .widget-chart { margin: 15px 0; }
        .widget-actions { display: flex; justify-content: flex-end; gap: 5px; }
    </style>
    
    <script>
    $(document).ready(function() {
        // Initialize chart if enabled
        <?php if ($showGraphs): ?>
        initYourWidgetChart();
        <?php endif; ?>
        
        // Auto-refresh
        setInterval(refreshYourWidget, <?php echo $refreshInterval * 1000; ?>);
    });
    
    function refreshYourWidget() {
        $.get('yourwidget.php?action=refresh', function(data) {
            $('.stat-value').first().text(data.total);
        });
    }
    
    <?php if ($showGraphs): ?>
    function initYourWidgetChart() {
        var ctx = document.getElementById('yourwidget-chart').getContext('2d');
        new Chart(ctx, {
            type: 'line',
            data: {
                labels: <?php echo json_encode($data['labels']); ?>,
                datasets: [{
                    label: 'Activity',
                    data: <?php echo json_encode($data['values']); ?>,
                    borderColor: '#007bff',
                    fill: false,
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: { y: { beginAtZero: true } }
            }
        });
    }
    <?php endif; ?>
    </script>
    <?php
}

function yourwidget_fetch_data()
{
    // Fetch data from database
    $data = [
        'total' => 0,
        'active' => 0,
        'pending' => 0,
        'labels' => ['Mon', 'Tue', 'Wed', 'Thu', 'Fri'],
        'values' => [12, 19, 15, 8, 20],
    ];
    
    // Use Capsule for database queries
    $capsule = Capsule::table('tblhosting')
        ->selectRaw('COUNT(*) as total')
        ->whereIn('domainstatus', ['Active', 'Suspended', 'Terminated'])
        ->first();
    
    $data['total'] = $capsule->total ?? 0;
    
    return $data;
}
```

## AJAX Refresh Handler

```php
<?php
// In a separate file for AJAX handling
if (defined('WHMCS')) {
    // Handle AJAX requests
    $action = $_GET['action'] ?? '';
    
    switch ($action) {
        case 'refresh':
            header('Content-Type: application/json');
            $data = yourwidget_fetch_data();
            echo json_encode($data);
            break;
            
        case 'details':
            $id = (int)($_GET['id'] ?? 0);
            $details = yourwidget_get_details($id);
            echo json_encode($details);
            break;
    }
}
```

## Configuration Settings

```php
function yourwidget_settings()
{
    return [
        'title' => [
            'FriendlyName' => 'Widget Title',
            'Type' => 'text',
            'Default' => 'Your Widget',
        ],
        'refresh_interval' => [
            'FriendlyName' => 'Auto Refresh',
            'Type' => 'dropdown',
            'Options' => [
                '0' => 'Disabled',
                '30' => 'Every 30 seconds',
                '60' => 'Every minute',
                '300' => 'Every 5 minutes',
            ],
        ],
        'show_graph' => [
            'FriendlyName' => 'Show Graph',
            'Type' => 'yesno',
            'Default' => '1',
        ],
        'data_source' => [
            'FriendlyName' => 'Data Source',
            'Type' => 'dropdown',
            'Options' => [
                'services' => 'Services',
                'invoices' => 'Invoices',
                'tickets' => 'Support Tickets',
            ],
        ],
    ];
}
```

## Step-by-Step Implementation

1. Create widget file in `/modules/widgets/`
2. Implement meta function for settings
3. Implement output function with HTML/PHP
4. Add CSS styling inline
5. Add JavaScript for interactivity
6. Create AJAX handlers for dynamic updates
7. Test widget in admin dashboard

## Implementation Checklist

- [ ] Create widget file in modules/widgets
- [ ] Implement meta function
- [ ] Implement settings function
- [ ] Implement output function with HTML
- [ ] Add inline CSS styling
- [ ] Add JavaScript for interactivity
- [ ] Implement AJAX refresh
- [ ] Add chart support if needed
- [ ] Test in admin dashboard