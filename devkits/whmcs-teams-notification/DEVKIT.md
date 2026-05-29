# WHMCS Microsoft Teams Notification Provider - DEVKIT

## Module Information
- **Name**: Microsoft Teams Notifications
- **Version**: 1.0.0
- **Type**: Notification Provider
- **Description**: Send WHMCS notifications to Microsoft Teams channels

## Installation
1. Copy to `/modules/notifications/teams/`
2. Activate via WHMCS Admin > Configuration > Notification Channels

## teams.php
```php
<?php
/**
 * WHMCS Microsoft Teams Notification Provider
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function teams_config()
{
    return [
        'name' => 'Microsoft Teams',
        'description' => 'Send notifications to Microsoft Teams channels',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'webhook_url' => [
                'FriendlyName' => 'Webhook URL',
                'Type' => 'text',
                'Size' => '100',
                'Description' => 'Teams Incoming Webhook URL'
            ],
            'theme_color' => [
                'FriendlyName' => 'Theme Color',
                'Type' => 'text',
                'Size' => '10',
                'Default' => '0078D4',
                'Description' => 'Hex color for adaptive cards'
            ],
            'mention_group' => [
                'FriendlyName' => 'Mention Group',
                'Type' => 'text',
                'Size' => '50',
                'Description' => 'Group name to mention for critical alerts'
            ],
            'severity_colors' => [
                'FriendlyName' => 'Severity Colors',
                'Type' => 'textarea',
                'Description' => 'JSON: {"critical":"FF0000","warning":"FFA500","info":"0078D4","success":"00FF00"}'
            ]
        ]
    ];
}

function teams_send($params)
{
    $webhookUrl = $params['webhook_url'];
    $themeColor = $params['theme_color'] ?? '0078D4';
    
    $title = $params['title'] ?? 'WHMCS Notification';
    $message = $params['message'] ?? '';
    $severity = $params['severity'] ?? 'info';
    
    // Map severity to colors
    $severityColors = json_decode($params['severity_colors'] ?? '{}', true) ?: [
        'critical' => 'FF0000',
        'warning' => 'FFA500',
        'info' => '0078D4',
        'success' => '00FF00'
    ];
    
    $cardColor = $severityColors[$severity] ?? $severityColors['info'];
    
    // Build Adaptive Card payload
    $payload = [
        '@type' => 'Message',
        '@context' => 'http://schema.org/extensions',
        'themeColor' => $cardColor,
        'summary' => $title,
        'sections' => [
            [
                'activityTitle' => $title,
                'activitySubtitle' => 'WHMCS Notification',
                'facts' => [],
                'text' => $message
            ]
        ]
    ];
    
    // Add fields
    if (!empty($params['fields'])) {
        foreach ($params['fields'] as $field) {
            $payload['sections'][0]['facts'][] = [
                'name' => $field['title'] ?? '',
                'value' => $field['value'] ?? ''
            ];
        }
    }
    
    // Add mention for critical alerts
    if (!empty($params['mention_group']) && in_array($severity, ['critical', 'warning'])) {
        $payload['sections'][0]['activitySubtitle'] .= ' - @' . $params['mention_group'];
    }
    
    // Send to Teams
    $ch = curl_init($webhookUrl);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    if ($httpCode >= 200 && $httpCode < 300) {
        return ['success' => true];
    }
    
    return [
        'success' => false,
        'error' => 'Failed to send notification'
    ];
}

function teams_preview($params)
{
    return teams_send([
        'webhook_url' => $params['webhook_url'],
        'theme_color' => $params['theme_color'] ?? '0078D4',
        'title' => 'Test Notification',
        'message' => 'This is a test notification from WHMCS',
        'severity' => 'info',
        'fields' => [
            ['title' => 'Test Field', 'value' => 'Test Value']
        ]
    ]);
}
```

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('OrderPlaced', 1, function($vars) {
    send_teams_notification([
        'title' => 'New Order #' . $vars['orderid'],
        'message' => 'Customer placed an order for $' . $vars['amount'],
        'severity' => 'success',
        'fields' => [
            ['title' => 'Order ID', 'value' => $vars['orderid']],
            ['title' => 'Amount', 'value' => '$' . $vars['amount']]
        ]
    ]);
});

add_hook('InvoicePaid', 1, function($vars) {
    send_teams_notification([
        'title' => 'Payment Received',
        'message' => 'Invoice #' . $vars['invoiceid'] . ' paid',
        'severity' => 'success',
        'fields' => [
            ['title' => 'Amount', 'value' => '$' . $vars['amount']]
        ]
    ]);
});

add_hook('TicketOpen', 1, function($vars) {
    send_teams_notification([
        'title' => 'New Support Ticket',
        'message' => $vars['subject'] ?? 'No subject',
        'severity' => 'warning',
        'fields' => [
            ['title' => 'Ticket ID', 'value' => $vars['ticketid']],
            ['title' => 'Priority', 'value' => $vars['priority'] ?? 'Medium']
        ]
    ]);
});

add_hook('ServiceSuspended', 1, function($vars) {
    send_teams_notification([
        'title' => 'Service Suspended',
        'message' => 'Service #' . $vars['serviceid'] . ' suspended',
        'severity' => 'warning'
    ]);
});

add_hook('ServiceTerminated', 1, function($vars) {
    send_teams_notification([
        'title' => 'Service Terminated',
        'message' => 'Service #' . $vars['serviceid'] . ' terminated',
        'severity' => 'critical'
    ]);
});

add_hook('DailyCronJob', 1, function($vars) {
    // Daily summary notification
    $stats = [
        'orders' => TeamsHelper::getTodayOrders(),
        'revenue' => TeamsHelper::getTodayRevenue(),
        'tickets' => TeamsHelper::getOpenTickets()
    ];
    
    send_teams_notification([
        'title' => 'Daily Summary',
        'message' => 'Orders: ' . $stats['orders'] . ', Revenue: $' . $stats['revenue'] . ', Open Tickets: ' . $stats['tickets'],
        'severity' => 'info'
    ]);
});

function send_teams_notification($data)
{
    $config = get_config('teams');
    
    if (empty($config['webhook_url'])) {
        return false;
    }
    
    return teams_send([
        'webhook_url' => $config['webhook_url'],
        'theme_color' => $config['theme_color'] ?? '0078D4',
        'mention_group' => $config['mention_group'] ?? '',
        'severity_colors' => $config['severity_colors'] ?? '{}',
        'title' => $data['title'],
        'message' => $data['message'],
        'severity' => $data['severity'] ?? 'info',
        'fields' => $data['fields'] ?? []
    ]);
}

class TeamsHelper
{
    public static function getTodayOrders()
    {
        $result = full_query("SELECT COUNT(*) as count FROM " . TABLE_PREFIX . "tblorders WHERE date = CURDATE()");
        $row = mysql_fetch_array($result);
        return $row['count'] ?? 0;
    }
    
    public static function getTodayRevenue()
    {
        $result = full_query("SELECT COALESCE(SUM(amountin), 0) as revenue FROM " . TABLE_PREFIX . "tblaccounts WHERE date = CURDATE()");
        $row = mysql_fetch_array($result);
        return number_format($row['revenue'] ?? 0, 2);
    }
    
    public static function getOpenTickets()
    {
        $result = full_query("SELECT COUNT(*) as count FROM " . TABLE_PREFIX . "tbltickets WHERE status NOT IN ('Closed', 'Resolved')");
        $row = mysql_fetch_array($result);
        return $row['count'] ?? 0;
    }
}
```