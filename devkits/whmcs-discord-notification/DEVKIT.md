# WHMCS Discord Notification Provider - DEVKIT

## Module Information
- **Name**: Discord Notifications
- **Version**: 1.0.0
- **Type**: Notification Provider
- **Description**: Send WHMCS notifications to Discord channels

## Installation
1. Copy to `/modules/notifications/discord/`
2. Activate via WHMCS Admin > Configuration > Notification Channels

## discord.php
```php
<?php
/**
 * WHMCS Discord Notification Provider
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function discord_config()
{
    return [
        'name' => 'Discord',
        'description' => 'Send notifications to Discord channels',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'webhook_url' => [
                'FriendlyName' => 'Webhook URL',
                'Type' => 'text',
                'Size' => '100',
                'Description' => 'Discord webhook URL'
            ],
            'username' => [
                'FriendlyName' => 'Bot Username',
                'Type' => 'text',
                'Size' => '50',
                'Default' => 'WHMCS Bot'
            ],
            'avatar_url' => [
                'FriendlyName' => 'Avatar URL',
                'Type' => 'text',
                'Size' => '100',
                'Description' => 'Bot avatar image URL'
            ],
            'mention_role_id' => [
                'FriendlyName' => 'Mention Role ID',
                'Type' => 'text',
                'Size' => '50',
                'Description' => 'Discord role ID to mention for alerts'
            ],
            'embed_color' => [
                'FriendlyName' => 'Embed Color',
                'Type' => 'text',
                'Size' => '10',
                'Default' => '3447003',
                'Description' => 'Hex color for embeds (e.g., 3447003 for blue)'
            ]
        ]
    ];
}

function discord_send($params)
{
    $webhookUrl = $params['webhook_url'];
    $username = $params['username'] ?? 'WHMCS Bot';
    $avatarUrl = $params['avatar_url'] ?? '';
    
    $title = $params['title'] ?? 'WHMCS Notification';
    $message = $params['message'] ?? '';
    $severity = $params['severity'] ?? 'info';
    
    // Map severity to colors
    $colors = [
        'critical' => 15158332,  // Red
        'warning' => 15105570,  // Orange
        'info' => 3447003,      // Blue
        'success' => 3066993    // Green
    ];
    
    $embedColor = $params['embed_color'] ?? $colors[$severity] ?? $colors['info'];
    
    // Build Discord embed
    $embed = [
        'title' => $title,
        'description' => $message,
        'color' => (int)$embedColor,
        'footer' => [
            'text' => 'WHMCS Notification System'
        ],
        'timestamp' => date('c')
    ];
    
    // Add fields
    if (!empty($params['fields'])) {
        $embed['fields'] = [];
        foreach ($params['fields'] as $field) {
            $embed['fields'][] = [
                'name' => $field['title'] ?? '',
                'value' => $field['value'] ?? '',
                'inline' => $field['short'] ?? true
            ];
        }
    }
    
    // Build payload
    $payload = [
        'username' => $username,
        'embeds' => [$embed]
    ];
    
    if (!empty($avatarUrl)) {
        $payload['avatar_url'] = $avatarUrl;
    }
    
    // Add role mention if configured
    if (!empty($params['mention_role_id']) && in_array($severity, ['critical', 'warning'])) {
        $payload['content'] = '<@&' . $params['mention_role_id'] . '>';
    }
    
    // Send to Discord
    $ch = curl_init($webhookUrl);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    if ($httpCode == 204 || $httpCode == 200) {
        return ['success' => true];
    }
    
    return [
        'success' => false,
        'error' => 'Failed to send notification'
    ];
}

function discord_preview($params)
{
    return discord_send([
        'webhook_url' => $params['webhook_url'],
        'username' => $params['username'] ?? 'WHMCS Bot',
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

add_hook('OrderPlaced', 1, function($vars) {
    send_discord_notification([
        'title' => 'New Order #' . $vars['orderid'],
        'message' => 'Customer placed an order for $' . $vars['amount'],
        'severity' => 'success',
        'fields' => [
            ['title' => 'Order ID', 'value' => $vars['orderid'], 'short' => true],
            ['title' => 'Amount', 'value' => '$' . $vars['amount'], 'short' => true]
        ]
    ]);
});

add_hook('InvoicePaid', 1, function($vars) {
    send_discord_notification([
        'title' => 'Payment Received',
        'message' => 'Invoice #' . $vars['invoiceid'] . ' paid: $' . $vars['amount'],
        'severity' => 'success',
        'fields' => [
            ['title' => 'Invoice', 'value' => $vars['invoiceid'], 'short' => true],
            ['title' => 'Amount', 'value' => '$' . $vars['amount'], 'short' => true]
        ]
    ]);
});

add_hook('TicketOpen', 1, function($vars) {
    send_discord_notification([
        'title' => 'New Support Ticket',
        'message' => $vars['subject'] ?? 'No subject',
        'severity' => 'warning',
        'fields' => [
            ['title' => 'Ticket ID', 'value' => $vars['ticketid'], 'short' => true],
            ['title' => 'Priority', 'value' => $vars['priority'] ?? 'Medium', 'short' => true]
        ]
    ]);
});

add_hook('ServiceSuspended', 1, function($vars) {
    send_discord_notification([
        'title' => 'Service Suspended',
        'message' => 'Service #' . $vars['serviceid'] . ' suspended',
        'severity' => 'warning'
    ]);
});

add_hook('ServiceTerminated', 1, function($vars) {
    send_discord_notification([
        'title' => 'Service Terminated',
        'message' => 'Service #' . $vars['serviceid'] . ' terminated',
        'severity' => 'critical'
    ]);
});

add_hook('InvoiceCreated', 1, function($vars) {
    if ($vars['total'] > 100) {
        send_discord_notification([
            'title' => 'Large Invoice Created',
            'message' => 'Invoice #' . $vars['invoiceid'] . ' for $' . $vars['total'],
            'severity' => 'info'
        ]);
    }
});

function send_discord_notification($data)
{
    $config = get_config('discord');
    
    if (empty($config['webhook_url'])) {
        return false;
    }
    
    return discord_send([
        'webhook_url' => $config['webhook_url'],
        'username' => $config['username'] ?? 'WHMCS Bot',
        'avatar_url' => $config['avatar_url'] ?? '',
        'mention_role_id' => $config['mention_role_id'] ?? '',
        'title' => $data['title'],
        'message' => $data['message'],
        'severity' => $data['severity'] ?? 'info',
        'fields' => $data['fields'] ?? []
    ]);
}
```