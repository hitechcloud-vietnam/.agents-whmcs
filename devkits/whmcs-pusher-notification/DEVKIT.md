# WHMCS Pusher Notification Provider - DEVKIT

## Module Information
- **Name**: Pusher Notifications
- **Version**: 1.0.0
- **Type**: Notification Provider
- **Description**: Real-time notifications via Pusher Beams

## Installation
1. Copy to `/modules/notifications/pusher/`
2. Activate via WHMCS Admin

## pusher.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function pusher_config()
{
    return [
        'name' => 'Pusher',
        'description' => 'Real-time notifications via Pusher',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'app_id' => ['FriendlyName' => 'App ID', 'Type' => 'text', 'Size' => '50'],
            'app_key' => ['FriendlyName' => 'App Key', 'Type' => 'password', 'Size' => '80'],
            'app_secret' => ['FriendlyName' => 'App Secret', 'Type' => 'password', 'Size' => '80'],
            'cluster' => ['FriendlyName' => 'Cluster', 'Type' => 'text', 'Size' => '20', 'Default' => 'mt1'],
            'instance_id' => ['FriendlyName' => 'Instance ID (Beams)', 'Type' => 'text', 'Size' => '80']
        ]
    ];
}

function pusher_send($params)
{
    $appId = $params['app_id'];
    $appKey = $params['app_key'];
    $appSecret = $params['app_secret'];
    $cluster = $params['cluster'] ?? 'mt1';
    
    $title = $params['title'] ?? 'WHMCS Notification';
    $message = $params['message'] ?? '';
    
    $payload = [
        'name' => 'notification',
        'data' => [
            'title' => $title,
            'body' => $message,
            'severity' => $params['severity'] ?? 'info',
            'timestamp' => time()
        ],
        'channels' => ['whmcs-notifications']
    ];
    
    $authTimestamp = time();
    $authVersion = '1.0';
    
    $stringToSign = 'POST' . "\n" . '/apps/' . $appId . '/events' . "\n" . 'auth_key=' . $appKey . '&auth_timestamp=' . $authTimestamp . '&auth_version=' . $authVersion;
    $signature = hash_hmac('sha256', $stringToSign, $appSecret);
    
    $ch = curl_init('https://api-' . $cluster . '.pusherapps.com/apps/' . $appId . '/events');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json',
        'Authorization: NativeApp key=' . $appKey . ', signature=' . $signature . ', timestamp=' . $authTimestamp
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    return isset($result['status']) ? ['success' => true] : ['success' => false, 'error' => 'Failed'];
}

function pusher_preview($params)
{
    return pusher_send([
        'app_id' => $params['app_id'],
        'app_key' => $params['app_key'],
        'app_secret' => $params['app_secret'],
        'cluster' => $params['cluster'],
        'title' => 'Test Notification',
        'message' => 'This is a test notification from WHMCS',
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
    pusher_send(['title' => 'New Order', 'message' => 'Order #' . $vars['orderid'] . ' placed', 'severity' => 'success']);
});

add_hook('InvoicePaid', 1, function($vars) {
    pusher_send(['title' => 'Payment', 'message' => 'Invoice #' . $vars['invoiceid'] . ' paid', 'severity' => 'success']);
});

add_hook('TicketOpen', 1, function($vars) {
    pusher_send(['title' => 'New Ticket', 'message' => $vars['subject'] ?? 'New ticket', 'severity' => 'warning']);
});
```