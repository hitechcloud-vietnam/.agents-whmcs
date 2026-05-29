# WHMCS Notification Provider From Scratch Workflow

## Description
Create a custom notification provider module for WHMCS from scratch.

## Prerequisites
- WHMCS 7.2+
- PHP 8.1+
- Notification service API (Slack, Discord, Teams, etc.)

## Module Structure
```
modules/notifications/
└── clicodes_slack/
    ├── clicodes_slack.php           # Main module
    ├── templates/
    │   ├── ticket_opened.tpl
    │   └── invoice_paid.tpl
    └── README.md
```

## Steps

### Step 1: Create Module Directory
```bash
mkdir -p /var/www/whmcs/modules/notifications/clicodes_slack
mkdir -p /var/www/whmcs/modules/notifications/clicodes_slack/templates
```

### Step 2: Create Main Notification Module
```php
<?php
/**
 * WHMCS Notification Provider - Slack
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Exception\Fatal;
use WHMCS\Module\Notification\DescriptionTrait;

/**
 * Define module metadata
 */
function clicodes_slack_MetaData()
{
    return [
        'DisplayName' => 'Slack Notification',
        'Description' => 'Send notifications to Slack channels.',
        'APIVersion' => '1.0',
    ];
}

/**
 * Define configuration options
 */
function clicodes_slack_config()
{
    return [
        'webhookUrl' => [
            'Type' => 'text',
            'FriendlyName' => 'Webhook URL',
            'Description' => 'Slack Incoming Webhook URL',
        ],
        'channel' => [
            'Type' => 'text',
            'FriendlyName' => 'Channel',
            'Description' => 'Default channel (leave empty for webhook default)',
        ],
        'username' => [
            'Type' => 'text',
            'FriendlyName' => 'Bot Username',
            'Description' => 'Bot name for notifications',
            'Default' => 'WHMCS Bot',
        ],
        'icon_emoji' => [
            'Type' => 'text',
            'FriendlyName' => 'Icon',
            'Description' => 'Emoji icon (e.g., :bell:)',
            'Default' => ':bell:',
        ],
        'color' => [
            'Type' => 'dropdown',
            'FriendlyName' => 'Accent Color',
            'Options' => [
                '#36a64f' => 'Green (Success)',
                '#ff0000' => 'Red (Error)',
                '#0074D9' => 'Blue (Info)',
                '#FFDC00' => 'Yellow (Warning)',
            ],
            'Default' => '#0074D9',
        ],
        'enabledEvents' => [
            'Type' => 'checkbox',
            'FriendlyName' => 'Enabled Events',
            'Inputs' => [
                'InvoicePaid' => 'Invoice Paid',
                'InvoiceCreated' => 'Invoice Created',
                'TicketOpened' => 'Ticket Opened',
                'TicketReplied' => 'Ticket Replied',
                'OrderPlaced' => 'Order Placed',
                'OrderProvisioned' => 'Order Provisioned',
                'ClientRegistration' => 'New Client',
                'ModuleSuspended' => 'Service Suspended',
            ],
        ],
    ];
}

/**
 * Send notification
 */
function clicodes_slack_sendNotification($hookParams)
{
    $config = $hookParams['moduleConfiguration'];
    $event = $hookParams['event'];
    
    // Check if event is enabled
    $enabledEvents = $config['enabledEvents'] ?? [];
    if (!empty($enabledEvents) && !in_array($event, $enabledEvents)) {
        return ['success' => true]; // Skip disabled events
    }
    
    $webhookUrl = $config['webhookUrl'];
    if (empty($webhookUrl)) {
        throw new Fatal('Slack webhook URL not configured');
    }
    
    $payload = clicodes_slack_buildPayload($hookParams, $config);
    
    $ch = curl_init($webhookUrl);
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode($payload),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
        CURLOPT_TIMEOUT => 10,
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    if ($httpCode !== 200) {
        throw new Fatal('Slack notification failed: HTTP ' . $httpCode);
    }
    
    return ['success' => true];
}

/**
 * Build Slack message payload
 */
function clicodes_slack_buildPayload($hookParams, $config)
{
    $event = $hookParams['event'];
    $data = $hookParams['notification'];
    
    $username = $config['username'] ?? 'WHMCS';
    $icon = $config['icon_emoji'] ?? ':bell:';
    $color = $config['color'] ?? '#0074D9';
    
    $fields = [];
    
    // Build fields based on event type
    switch ($event) {
        case 'InvoicePaid':
            $title = 'Invoice Paid';
            $fields[] = ['title' => 'Invoice ID', 'value' => $data['invoice_id'] ?? '', 'short' => true];
            $fields[] = ['title' => 'Amount', 'value' => $data['amount'] ?? '', 'short' => true];
            $fields[] = ['title' => 'Client', 'value' => $data['client_name'] ?? '', 'short' => true];
            $color = '#36a64f';
            break;
            
        case 'TicketOpened':
            $title = 'New Support Ticket';
            $fields[] = ['title' => 'Ticket ID', 'value' => $data['ticket_id'] ?? '', 'short' => true];
            $fields[] = ['title' => 'Subject', 'value' => substr($data['subject'] ?? '', 0, 100), 'short' => false];
            $fields[] = ['title' => 'Priority', 'value' => $data['priority'] ?? '', 'short' => true];
            $fields[] = ['title' => 'Client', 'value' => $data['client_name'] ?? '', 'short' => true];
            $color = '#FFDC00';
            break;
            
        case 'OrderPlaced':
            $title = 'New Order';
            $fields[] = ['title' => 'Order ID', 'value' => $data['order_id'] ?? '', 'short' => true];
            $fields[] = ['title' => 'Amount', 'value' => $data['total'] ?? '', 'short' => true];
            $fields[] = ['title' => 'Products', 'value' => $data['products'] ?? '', 'short' => false];
            $color = '#36a64f';
            break;
            
        default:
            $title = 'WHMCS Notification';
            $fields[] = ['title' => 'Event', 'value' => $event, 'short' => true];
            $fields[] = ['title' => 'Time', 'value' => date('Y-m-d H:i:s'), 'short' => true];
    }
    
    $payload = [
        'username' => $username,
        'icon_emoji' => $icon,
        'channel' => $config['channel'] ?: null,
        'attachments' => [
            [
                'color' => $color,
                'title' => $title,
                'fields' => $fields,
                'footer' => 'WHMCS',
                'ts' => time(),
            ],
        ],
    ];
    
    return $payload;
}

/**
 * Define notification parameters
 */
function clicodes_slack_notificationParameters()
{
    return [];
}
```

### Step 3: Install and Configure
```bash
# Copy module
cp -r clicodes_slack /var/www/whmcs/modules/notifications/

# Set permissions
chown -R www-data:www-data /var/www/whmcs/modules/notifications/clicodes_slack
chmod -R 755 /var/www/whmcs/modules/notifications/clicodes_slack

# Configure in WHMCS Admin
# Go to: Configuration > System Settings > Notification Templates
# Add new notification channel
# Enter Webhook URL from Slack
```

## Testing
```php
// Test notification
$params = [
    'event' => 'InvoicePaid',
    'notification' => [
        'invoice_id' => '12345',
        'amount' => '$99.99',
        'client_name' => 'Test Client',
    ],
    'moduleConfiguration' => [
        'webhookUrl' => 'https://hooks.slack.com/...',
        'username' => 'WHMCS Bot',
        'icon_emoji' => ':bell:',
        'color' => '#36a64f',
    ],
];

$result = clicodes_slack_sendNotification($params);
```

## Tags
- notification
- slack
- integration
- module-development