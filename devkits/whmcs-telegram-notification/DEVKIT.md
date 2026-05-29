# WHMCS Telegram Notification Provider - DEVKIT

## Module Information
- **Name**: Telegram Notifications
- **Version**: 1.0.0
- **Type**: Notification Provider
- **Description**: Send WHMCS notifications via Telegram bot

## Installation
1. Copy to `/modules/notifications/telegram/`
2. Activate via WHMCS Admin > Configuration > Notification Channels

## telegram.php
```php
<?php
/**
 * WHMCS Telegram Notification Provider
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function telegram_config()
{
    return [
        'name' => 'Telegram',
        'description' => 'Send notifications via Telegram Bot',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'bot_token' => [
                'FriendlyName' => 'Bot Token',
                'Type' => 'password',
                'Size' => '80',
                'Description' => 'Telegram Bot Token from @BotFather'
            ],
            'chat_id' => [
                'FriendlyName' => 'Chat ID',
                'Type' => 'text',
                'Size' => '50',
                'Description' => 'Telegram Chat ID or channel username'
            ],
            'parse_mode' => [
                'FriendlyName' => 'Parse Mode',
                'Type' => 'dropdown',
                'Options' => [
                    'Markdown' => 'Markdown',
                    'HTML' => 'HTML',
                    'None' => 'None'
                ],
                'Default' => 'Markdown'
            ],
            'disable_web_preview' => [
                'FriendlyName' => 'Disable Link Preview',
                'Type' => 'yesno',
                'Description' => 'Disable link preview in messages'
            ],
            'message_template' => [
                'FriendlyName' => 'Message Template',
                'Type' => 'textarea',
                'Description' => 'Custom message template with placeholders'
            ]
        ]
    ];
}

function telegram_send($params)
{
    $botToken = $params['bot_token'];
    $chatId = $params['chat_id'];
    $parseMode = $params['parse_mode'] ?? 'Markdown';
    
    $title = $params['title'] ?? 'WHMCS Notification';
    $message = $params['message'] ?? '';
    $severity = $params['severity'] ?? 'info';
    
    // Build emoji based on severity
    $emojis = [
        'critical' => ':rotating_light:',
        'warning' => ':warning:',
        'info' => ':information_source:',
        'success' => ':white_check_mark:'
    ];
    
    $emoji = $emojis[$severity] ?? $emojis['info'];
    
    // Format message
    $formattedMessage = "*" . $emoji . " " . $title . "*\n\n";
    $formattedMessage .= $message . "\n\n";
    $formattedMessage .= "_Sent from WHMCS_";
    
    // Add fields if provided
    if (!empty($params['fields'])) {
        $formattedMessage .= "\n\n";
        foreach ($params['fields'] as $field) {
            $fieldTitle = $field['title'] ?? '';
            $fieldValue = $field['value'] ?? '';
            
            if ($parseMode == 'HTML') {
                $formattedMessage .= "<b>" . htmlspecialchars($fieldTitle) . ":</b> " . htmlspecialchars($fieldValue) . "\n";
            } else {
                $formattedMessage .= "*" . $fieldTitle . ":* " . $fieldValue . "\n";
            }
        }
    }
    
    // Prepare API request
    $url = 'https://api.telegram.org/bot' . $botToken . '/sendMessage';
    
    $postData = [
        'chat_id' => $chatId,
        'text' => $formattedMessage,
        'parse_mode' => $parseMode == 'None' ? '' : $parseMode,
        'disable_web_page_preview' => $params['disable_web_preview'] ?? false
    ];
    
    // Send message
    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, $postData);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['ok']) && $result['ok']) {
        return ['success' => true];
    }
    
    return [
        'success' => false,
        'error' => $result['description'] ?? 'Failed to send notification'
    ];
}

function telegram_preview($params)
{
    return telegram_send([
        'bot_token' => $params['bot_token'],
        'chat_id' => $params['chat_id'],
        'parse_mode' => $params['parse_mode'] ?? 'Markdown',
        'title' => 'Test Notification',
        'message' => 'This is a test notification from WHMCS',
        'severity' => 'info',
        'fields' => [
            ['title' => 'Test Field', 'value' => 'Test Value']
        ]
    ]);
}

function telegram_set_webhook($params)
{
    $botToken = $params['bot_token'];
    $webhookUrl = $params['webhook_url'];
    
    $ch = curl_init('https://api.telegram.org/bot' . $botToken . '/setWebhook');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, ['url' => $webhookUrl]);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    return json_decode($response, true);
}

function telegram_get_updates($params)
{
    $botToken = $params['bot_token'];
    
    $ch = curl_init('https://api.telegram.org/bot' . $botToken . '/getUpdates');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    return json_decode($response, true);
}
```

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('OrderPlaced', 1, function($vars) {
    send_telegram_notification([
        'title' => 'New Order #' . $vars['orderid'],
        'message' => 'Customer placed an order',
        'severity' => 'success',
        'fields' => [
            ['title' => 'Order ID', 'value' => $vars['orderid']],
            ['title' => 'Amount', 'value' => '$' . $vars['amount']]
        ]
    ]);
});

add_hook('InvoicePaid', 1, function($vars) {
    send_telegram_notification([
        'title' => 'Payment Received',
        'message' => 'Invoice #' . $vars['invoiceid'] . ' paid',
        'severity' => 'success',
        'fields' => [
            ['title' => 'Amount', 'value' => '$' . $vars['amount']]
        ]
    ]);
});

add_hook('TicketOpen', 1, function($vars) {
    send_telegram_notification([
        'title' => 'New Ticket #' . $vars['ticketid'],
        'message' => $vars['subject'] ?? 'No subject',
        'severity' => 'warning',
        'fields' => [
            ['title' => 'Priority', 'value' => $vars['priority'] ?? 'Medium']
        ]
    ]);
});

add_hook('ServiceSuspended', 1, function($vars) {
    send_telegram_notification([
        'title' => 'Service Suspended',
        'message' => 'Service #' . $vars['serviceid'] . ' suspended',
        'severity' => 'warning'
    ]);
});

add_hook('ServiceTerminated', 1, function($vars) {
    send_telegram_notification([
        'title' => 'Service Terminated',
        'message' => 'Service #' . $vars['serviceid'] . ' terminated',
        'severity' => 'critical'
    ]);
});

add_hook('AdminLogin', 1, function($vars) {
    send_telegram_notification([
        'title' => 'Admin Login',
        'message' => 'Admin ' . $vars['admin_username'] . ' logged in',
        'severity' => 'info',
        'fields' => [
            ['title' => 'IP Address', 'value' => $_SERVER['REMOTE_ADDR'] ?? 'Unknown']
        ]
    ]);
});

function send_telegram_notification($data)
{
    $config = get_config('telegram');
    
    if (empty($config['bot_token']) || empty($config['chat_id'])) {
        return false;
    }
    
    return telegram_send([
        'bot_token' => $config['bot_token'],
        'chat_id' => $config['chat_id'],
        'parse_mode' => $config['parse_mode'] ?? 'Markdown',
        'disable_web_preview' => $config['disable_web_preview'] ?? false,
        'title' => $data['title'],
        'message' => $data['message'],
        'severity' => $data['severity'] ?? 'info',
        'fields' => $data['fields'] ?? []
    ]);
}
```