# WHMCS Client Dashboard - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_dashboard_config() {
    return [
        'name' => 'Client Dashboard',
        'description' => 'Enhanced client dashboard with widgets and analytics',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'show_widgets' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show dashboard widgets'],
            'show_analytics' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Show analytics'],
            'quick_actions' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable quick actions']
        ]
    ];
}

function whmcs_client_dashboard_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_dashboard_widgets` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `widget_key` VARCHAR(50) NOT NULL,
        `widget_name` VARCHAR(255) NOT NULL,
        `widget_type` VARCHAR(50) DEFAULT 'standard',
        `config` TEXT,
        `sort_order` INT(11) DEFAULT 0,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_dashboard_widget_settings` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `user_id` INT(11) NOT NULL,
        `widget_id` INT(11) NOT NULL,
        `position` INT(11) DEFAULT 0,
        `is_visible` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`),
        KEY `user_id` (`user_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    // Default widgets
    $widgets = [
        ['services', 'My Services', 'list', 1],
        ['invoices', 'Outstanding Invoices', 'summary', 2],
        ['tickets', 'Recent Tickets', 'list', 3],
        ['billing', 'Billing Summary', 'chart', 4],
        ['quick_actions', 'Quick Actions', 'buttons', 5]
    ];
    foreach ($widgets as $w) {
        insert_query('mod_dashboard_widgets', ['widget_key' => $w[0], 'widget_name' => $w[1], 'widget_type' => $w[2], 'sort_order' => $w[3]]);
    }
    return ['status' => 'success'];
}

function whmcs_client_dashboard_deactivate() { return ['status' => 'success']; }

function whmcs_client_dashboard_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Dashboard</h2>';
    
    $widgets = select_query('mod_dashboard_widgets', '*', '', 'sort_order');
    echo '<div class="panel panel-default">
          <div class="panel-heading">Active Widgets</div>
          <div class="panel-body">
          <table class="datatable">
          <thead><tr><th>Widget</th><th>Type</th><th>Order</th><th>Active</th></tr></thead>
          <tbody>';
    while ($w = mysql_fetch_array($widgets)) {
        echo '<tr><td>' . $w['widget_name'] . '</td>
              <td>' . $w['widget_type'] . '</td>
              <td>' . $w['sort_order'] . '</td>
              <td>' . ($w['is_active'] ? 'Yes' : 'No') . '</td></tr>';
    }
    echo '</tbody></table></div></div>';
}

add_hook('ClientAreaPageHome', 1, function($vars) {
    $uid = $_SESSION['uid'] ?? 0;
    if (!$uid) return [];
    
    // Get user's services
    $services = select_query('tblhosting', 'COUNT(*) as count', ['userid' => $uid, 'domainstatus' => 'Active']);
    $svcCount = mysql_fetch_array($services);
    
    // Get outstanding invoices
    $invoices = select_query('tblinvoices', 'SUM(total) as total', ['userid' => $uid, 'status' => 'Unpaid']);
    $invTotal = mysql_fetch_array($invoices);
    
    // Get open tickets
    $tickets = select_query('tbltickets', 'COUNT(*) as count', ['userid' => $uid, 'status' => 'Open']);
    $tktCount = mysql_fetch_array($tickets);
    
    return [
        'dashboard_widgets' => true,
        'services_count' => $svcCount['count'],
        'outstanding_balance' => $invTotal['total'] ?: 0,
        'open_tickets' => $tktCount['count']
    ];
});
```