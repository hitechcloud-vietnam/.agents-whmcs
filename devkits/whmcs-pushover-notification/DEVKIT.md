# WHMCS Pushover Notification Provider - DEVKIT

## Module Information
- **Name**: Pushover Notifications
- **Version**: 1.0.0
- **Type**: Notification Provider
- **Description**: Send push notifications via Pushover API

## Installation
1. Copy to `/modules/notifications/pushover/`
2. Activate via WHMCS Admin

## pushover.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function pushover_config()
{
    return [
        'name' => 'Pushover',
        'description' => 'Push notifications via Pushover',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'api_token' => ['FriendlyName' => 'API Token', 'Type' => 'password', 'Size' => '80', 'Description' => 'Pushover App Token'],
            'user_key' => ['FriendlyName' => 'User Key', 'Type' => 'text', 'Size' => '50', 'Description' => 'Pushover User Key'],
            'device' => ['FriendlyName' => 'Device', 'Type' => 'text', 'Size' => '50', 'Description' => 'Specific device (optional)'],
            'sound' => ['FriendlyName' => 'Sound', 'Type' => 'dropdown', 'Options' => ['pushover' => 'Default', 'bike' => 'Bike', 'bugle' => 'Bugle', 'cashregister' => 'Cash Register', 'classical' => 'Classical', 'echo' => 'Echo', 'gamelan' => 'Gamelan', 'incoming' => 'Incoming', 'intermission' => 'Intermission', 'magic' => 'Magic', 'mechanical' => 'Mechanical', 'persistent' => 'Persistent', 'pirate' => 'Pirate', 'siren' => 'Siren', 'tugboat' => 'Tugboat', 'none' => 'None']]
        ]
    ];
}

function pushover_send($params)
{
    $apiToken = $params['api_token'];
    $userKey = $params['user_key'];
    
    $title = $params['title'] ?? 'WHMCS';
    $message = $params['message'] ?? '';
    
    $severity = $params['severity'] ?? 'info';
    $priorities = ['critical' => 2, 'warning' => 1, 'info' => 0, 'success' => 0];
    $priority = $priorities[$severity] ?? 0;
    
    $data = [
        'token' => $apiToken,
        'user' => $userKey,
        'title' => $title,
        'message' => $message,
        'priority' => $priority,
        'timestamp' => time()
    ];
    
    if (!empty($params['device'])) {
        $data['device'] = $params['device'];
    }
    
    if (!empty($params['sound'])) {
        $data['sound'] = $params['sound'];
    }
    
    if ($priority == 2) {
        $data['retry'] = 60;
        $data['expire'] = 3600;
    }
    
    $ch = curl_init('https://api.pushover.net/1/messages.json');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, $data);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    return isset($result['status']) && $result['status'] == 1 ? ['success' => true] : ['success' => false, 'error' => $result['errors'][0] ?? 'Failed'];
}

function pushover_preview($params)
{
    return pushover_send([
        'api_token' => $params['api_token'],
        'user_key' => $params['user_key'],
        'title' => 'Test Notification',
        'message' => 'This is a test from WHMCS',
        'severity' => 'info'
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
    pushover_send(['title' => 'New Order', 'message' => 'Order #' . $vars['orderid'] . ' placed']);
});

add_hook('InvoicePaid', 1, function($vars) {
    pushover_send(['title' => 'Payment Received', 'message' => 'Invoice #' . $vars['invoiceid'] . ' paid: $' . $vars['amount']]);
});

add_hook('TicketOpen', 1, function($vars) {
    pushover_send(['title' => 'New Ticket', 'message' => $vars['subject'] ?? 'New support ticket', 'severity' => 'warning']);
});

add_hook('ServiceSuspended', 1, function($vars) {
    pushover_send(['title' => 'Service Suspended', 'message' => 'Service #' . $vars['serviceid'] . ' suspended', 'severity' => 'critical']);
});
```