# WHMCS Slack Notification Provider - DEVKIT

## Module Information
- **Name**: Slack Notifications
- **Version**: 1.0.0
- **Type**: Notification Provider
- **Description**: Send WHMCS notifications to Slack channels

## Installation
1. Copy to `/modules/notifications/slack/`
2. Activate via WHMCS Admin > Configuration > Notification Channels

## slack.php
```php
<?php
/**
 * WHMCS Slack Notification Provider
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function slack_config()
{
    return [
        'name' => 'Slack',
        'description' => 'Send notifications to Slack channels',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'webhook_url' => [
                'FriendlyName' => 'Webhook URL',
                'Type' => 'text',
                'Size' => '100',
                'Description' => 'Slack incoming webhook URL'
            ],
            'channel' => [
                'FriendlyName' => 'Default Channel',
                'Type' => 'text',
                'Size' => '50',
                'Description' => 'Default channel for notifications'
            ],
            'username' => [
                'FriendlyName' => 'Bot Username',
                'Type' => 'text',
                'Size' => '50',
                'Default' => 'WHMCS Bot',
                'Description' => 'Username for the bot'
            ],
            'icon_emoji' => [
                'FriendlyName' => 'Icon Emoji',
                'Type' => 'text',
                'Size' => '20',
                'Default' => ':robot_face:',
                'Description' => 'Emoji for bot icon'
            ],
            'mention_role' => [
                'FriendlyName' => 'Mention Role',
                'Type' => 'dropdown',
                'Options' => [
                    'none' => 'No mention',
                    'admin' => 'Admins only',
                    'all' => 'Everyone'
                ],
                'Default' => 'none'
            ]
        ]
    ];
}

function slack_send($params)
{
    $webhookUrl = $params['webhook_url'];
    $channel = $params['channel'] ?? '';
    $username = $params['username'] ?? 'WHMCS Bot';
    $iconEmoji = $params['icon_emoji'] ?? ':robot_face:';
    
    $title = $params['title'] ?? 'WHMCS Notification';
    $message = $params['message'] ?? '';
    $severity = $params['severity'] ?? 'info';
    
    // Build Slack message
    $payload = [
        'username' => $username,
        'icon_emoji' => $iconEmoji
    ];
    
    if (!empty($channel)) {
        $payload['channel'] = $channel;
    }
    
    // Color based on severity
    $colors = [
        'critical' => '#FF0000',
        'warning' => '#FFA500',
        'info' => '#36A64F',
        'success' => '#00FF00'
    ];
    
    // Build attachment
    $attachment = [
        'color' => $colors[$severity] ?? $colors['info'],
        'title' => $title,
        'text' => $message,
        'footer' => 'WHMCS',
        'ts' => time()
    ];
    
    // Add fields if provided
    if (!empty($params['fields'])) {
        $fields = [];
        foreach ($params['fields'] as $field) {
            $fields[] = [
                'title' => $field['title'] ?? '',
                'value' => $field['value'] ?? '',
                'short' => $field['short'] ?? true
            ];
        }
        $attachment['fields'] = $fields;
    }
    
    $payload['attachments'] = [$attachment];
    
    // Send to Slack
    $ch = curl_init($webhookUrl);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    if ($httpCode == 200) {
        return ['success' => true];
    }
    
    return [
        'success' => false,
        'error' => 'Failed to send notification'
    ];
}

function slack_preview($params)
{
    return slack_send([
        'webhook_url' => $params['webhook_url'],
        'title' => 'Test Notification',
        'message' => 'This is a test notification from WHMCS',
        'severity' => 'info',
        'fields' => [
            ['title' => 'Test Field', 'value' => 'Test Value', 'short' => true]
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

// Hook: New order placed
add_hook('OrderPlaced', 1, function($vars) {
    send_slack_notification([
        'title' => 'New Order #' . $vars['orderid'],
        'message' => 'Customer ' . $vars['customfield1'] ?? 'Unknown' . ' placed an order for $' . $vars['amount'],
        'severity' => 'success',
        'fields' => [
            ['title' => 'Order ID', 'value' => $vars['orderid'], 'short' => true],
            ['title' => 'Amount', 'value' => '$' . $vars['amount'], 'short' => true]
        ]
    ]);
});

// Hook: Invoice paid
add_hook('InvoicePaid', 1, function($vars) {
    send_slack_notification([
        'title' => 'Payment Received - Invoice #' . $vars['invoiceid'],
        'message' => 'Payment of $' . $vars['amount'] . ' received',
        'severity' => 'success',
        'fields' => [
            ['title' => 'Invoice', 'value' => $vars['invoiceid'], 'short' => true],
            ['title' => 'Amount', 'value' => '$' . $vars['amount'], 'short' => true]
        ]
    ]);
});

// Hook: New ticket created
add_hook('TicketOpen', 1, function($vars) {
    send_slack_notification([
        'title' => 'New Support Ticket #' . $vars['ticketid'],
        'message' => $vars['subject'] ?? 'No subject',
        'severity' => 'warning',
        'fields' => [
            ['title' => 'Ticket ID', 'value' => $vars['ticketid'], 'short' => true],
            ['title' => 'Priority', 'value' => $vars['priority'] ?? 'Medium', 'short' => true]
        ]
    ]);
});

// Hook: Service suspended
add_hook('ServiceSuspended', 1, function($vars) {
    send_slack_notification([
        'title' => 'Service Suspended',
        'message' => 'Service #' . $vars['serviceid'] . ' has been suspended',
        'severity' => 'warning',
        'fields' => [
            ['title' => 'Service ID', 'value' => $vars['serviceid'], 'short' => true],
            ['title' => 'Domain', 'value' => $vars['domain'] ?? 'N/A', 'short' => true]
        ]
    ]);
});

// Hook: Service terminated
add_hook('ServiceTerminated', 1, function($vars) {
    send_slack_notification([
        'title' => 'Service Terminated',
        'message' => 'Service #' . $vars['serviceid'] . ' has been terminated',
        'severity' => 'critical',
        'fields' => [
            ['title' => 'Service ID', 'value' => $vars['serviceid'], 'short' => true],
            ['title' => 'Domain', 'value' => $vars['domain'] ?? 'N/A', 'short' => true]
        ]
    ]);
});

// Hook: Domain expiring soon
add_hook('DomainTransferCompleted', 1, function($vars) {
    send_slack_notification([
        'title' => 'Domain Transferred',
        'message' => 'Domain ' . $vars['domain'] . ' has been transferred',
        'severity' => 'info',
        'fields' => [
            ['title' => 'Domain', 'value' => $vars['domain'], 'short' => true]
        ]
    ]);
});

// Hook: Admin login
add_hook('AdminLogin', 1, function($vars) {
    send_slack_notification([
        'title' => 'Admin Login',
        'message' => 'Admin ' . $vars['admin_username'] . ' logged in',
        'severity' => 'info',
        'fields' => [
            ['title' => 'Admin', 'value' => $vars['admin_username'], 'short' => true],
            ['title' => 'IP', 'value' => $_SERVER['REMOTE_ADDR'] ?? 'Unknown', 'short' => true]
        ]
    ]);
});

// Helper function
function send_slack_notification($data)
{
    $config = get_config('slack');
    
    if (empty($config['webhook_url'])) {
        return false;
    }
    
    return slack_send([
        'webhook_url' => $config['webhook_url'],
        'channel' => $config['channel'] ?? '',
        'username' => $config['username'] ?? 'WHMCS Bot',
        'icon_emoji' => $config['icon_emoji'] ?? ':robot_face:',
        'title' => $data['title'],
        'message' => $data['message'],
        'severity' => $data['severity'] ?? 'info',
        'fields' => $data['fields'] ?? []
    ]);
}
```