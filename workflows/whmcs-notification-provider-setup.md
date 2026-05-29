# WHMCS Notification Provider Setup

## Overview

This workflow guides you through creating a custom notification provider for WHMCS. Notification providers enable sending alerts through various channels like Slack, Discord, SMS, or custom APIs.

## Prerequisites

- WHMCS v8.0+
- PHP 7.4+
- API credentials for notification service
- Understanding of WHMCS notification system

## Step-by-Step Instructions

### Step 1: Create Notification Provider Structure

```
/modules/notifications/YourNotifier/
    ├── YourNotifier.php          # Main provider class
    ├── YourNotifierApi.php        # API communication
    └── lang/
        └── english.php
```

### Step 2: Create the Notification Provider Class

Create `YourNotifier.php`:

```php
<?php
/**
 * WHMCS Notification Provider - YourNotifier
 *
 * @copyright Copyright (c) 2024 Your Name
 */

namespace WHMCS\Notifications;

use WHMCS\Traits\NotificationProviderTrait;

class YourNotifier extends AbstractNotificationProvider
{
    use NotificationProviderTrait;

    /**
     * Provider display name.
     *
     * @var string
     */
    protected $name = 'Your Notifier';

    /**
     * Provider icon.
     *
     * @var string
     */
    protected $icon = 'fab fa-bell';

    /**
     * Default configuration.
     *
     * @return array
     */
    public function configuration()
    {
        return [
            'displayName' => [
                'Type' => 'text',
                'Name' => 'displayName',
                'Label' => 'Display Name',
                'Description' => 'How this integration is shown in WHMCS',
                'Value' => 'Your Notifier',
            ],
            'apiToken' => [
                'Type' => 'password',
                'Name' => 'apiToken',
                'Label' => 'API Token',
                'Description' => 'Your API token from the provider',
            ],
            'defaultChannel' => [
                'Type' => 'text',
                'Name' => 'defaultChannel',
                'Label' => 'Default Channel',
                'Description' => 'Default channel for notifications',
            ],
            'username' => [
                'Type' => 'text',
                'Name' => 'username',
                'Label' => 'Bot Username',
                'Description' => 'Username for the bot',
                'Value' => 'WHMCS Bot',
            ],
            'avatarUrl' => [
                'Type' => 'text',
                'Name' => 'avatarUrl',
                'Label' => 'Avatar URL',
                'Description' => 'URL to bot avatar image',
            ],
            'events' => [
                'Type' => 'checkbox',
                'Name' => 'events',
                'Label' => 'Notification Events',
                'Description' => 'Select which events trigger notifications',
                'Options' => [
                    'invoice_created' => 'Invoice Created',
                    'invoice_paid' => 'Invoice Paid',
                    'order_created' => 'New Order',
                    'service_created' => 'Service Activated',
                    'ticket_opened' => 'Ticket Opened',
                    'ticket_reply' => 'Ticket Reply',
                    'domain_renewal' => 'Domain Renewal Reminder',
                    'account_created' => 'New Account',
                ],
            ],
        ];
    }

    /**
     * Test connection to notification service.
     *
     * @param array $config
     * @return NotificationStatus
     */
    public function testConnection(array $config)
    {
        try {
            $api = new YourNotifierApi($config['apiToken']);

            $result = $api->sendTest([
                'channel' => $config['defaultChannel'] ?? 'general',
                'message' => 'WHMCS connection test successful!',
            ]);

            if ($result['success']) {
                return new NotificationStatus(true, 'Connection successful');
            }

            return new NotificationStatus(false, 'Failed to send test message');
        } catch (\Exception $e) {
            return new NotificationStatus(false, 'Connection failed: ' . $e->getMessage());
        }
    }

    /**
     * Send notification.
     *
     * @param Notification $notification
     * @return NotificationStatus
     */
    public function send(Notification $notification)
    {
        try {
            $config = $this->getConfig();
            $api = new YourNotifierApi($config['apiToken']);

            // Build message from notification
            $message = $this->buildMessage($notification);

            // Determine target channel
            $channel = $this->getChannel($notification);

            $result = $api->send([
                'channel' => $channel,
                'message' => $message['text'],
                'embed' => $message['embed'] ?? null,
                'username' => $config['username'] ?? 'WHMCS',
                'avatar_url' => $config['avatarUrl'] ?? null,
            ]);

            if ($result['success']) {
                return new NotificationStatus(true, 'Notification sent');
            }

            return new NotificationStatus(false, 'Failed to send notification');
        } catch (\Exception $e) {
            return new NotificationStatus(false, 'Error: ' . $e->getMessage());
        }
    }

    /**
     * Build notification message.
     *
     * @param Notification $notification
     * @return array
     */
    protected function buildMessage(Notification $notification)
    {
        $title = $notification->getTitle();
        $message = $notification->getMessage();
        $priority = $notification->getPriority();

        // Color based on priority
        $colorMap = [
            NotificationPriority::LOW => 0x808080,
            NotificationPriority::NORMAL => 0x3498db,
            NotificationPriority::HIGH => 0xf39c12,
            NotificationPriority::URGENT => 0xe74c3c,
        ];

        $embed = [
            'title' => $title,
            'description' => $message,
            'color' => $colorMap[$priority] ?? 0x3498db,
            'timestamp' => date('c'),
            'footer' => [
                'text' => 'WHMCS Notification',
            ],
        ];

        // Add fields based on notification type
        $fields = $notification->getFields();
        if (!empty($fields)) {
            foreach (array_slice($fields, 0, 10) as $field) {
                $embed['fields'][] = [
                    'name' => $field['title'] ?? '',
                    'value' => $field['value'] ?? '',
                    'inline' => $field['inline'] ?? false,
                ];
            }
        }

        // Add URL if present
        if ($notification->getUrl()) {
            $embed['url'] = $notification->getUrl();
        }

        return [
            'text' => $title . ': ' . $message,
            'embed' => $embed,
        ];
    }

    /**
     * Get target channel for notification.
     *
     * @param Notification $notification
     * @return string
     */
    protected function getChannel(Notification $notification)
    {
        $config = $this->getConfig();
        $event = $notification->getEvent();

        // Map events to channels
        $channelMap = [
            'invoice_paid' => 'payments',
            'order_created' => 'sales',
            'ticket_opened' => 'support',
            'ticket_reply' => 'support',
        ];

        return $channelMap[$event] ?? $config['defaultChannel'] ?? 'general';
    }

    /**
     * Determine if this notification should be sent.
     *
     * @param Notification $notification
     * @return bool
     */
    public function shouldSend(Notification $notification)
    {
        $config = $this->getConfig();

        // Check if event is enabled
        $events = $config['events'] ?? [];
        $event = $notification->getEvent();

        if (!empty($events) && !in_array($event, $events)) {
            return false;
        }

        return true;
    }
}
```

### Step 3: Create API Communication Class

Create `YourNotifierApi.php`:

```php
<?php

namespace YourNotifier;

class ApiClient
{
    private $apiToken;
    private $baseUrl = 'https://api.yourntifier.com/v1';

    public function __construct(string $apiToken)
    {
        $this->apiToken = $apiToken;
    }

    /**
     * Send notification message.
     *
     * @param array $data
     * @return array
     */
    public function send(array $data): array
    {
        $ch = curl_init($this->baseUrl . '/messages');

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiToken,
                'Content-Type: application/json',
            ],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $result = json_decode($response, true);

        if ($httpCode !== 200) {
            throw new \Exception($result['message'] ?? 'API request failed');
        }

        return [
            'success' => true,
            'message_id' => $result['id'] ?? null,
        ];
    }

    /**
     * Send test message.
     *
     * @param array $data
     * @return array
     */
    public function sendTest(array $data): array
    {
        return $this->send($data);
    }

    /**
     * Get channel information.
     *
     * @param string $channel
     * @return array
     */
    public function getChannel(string $channel): array
    {
        $ch = curl_init($this->baseUrl . '/channels/' . urlencode($channel));

        curl_setopt_array($ch, [
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiToken,
            ],
            CURLOPT_RETURNTRANSFER => true,
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }
}
```

### Step 4: Create Language File

Create `lang/english.php`:

```php
<?php

return [
    'yournotifier' => 'Your Notifier',
    'yournotifier_description' => 'Send notifications via Your Notifier',
];
```

### Step 5: Install and Configure

1. Upload to `/modules/notifications/YourNotifier/`
2. Go to Configuration > System > Notification Templates
3. Click "Add Integration"
4. Select "Your Notifier"
5. Enter configuration details
6. Test connection

## Expected Outcomes

- Notification provider appears in integration list
- Test connection verifies API access
- Notifications appear in configured channels
- Event filtering works correctly

## Testing Checklist

- [ ] Provider installs without errors
- [ ] Configuration fields save correctly
- [ ] Test connection succeeds
- [ ] Test notification sends
- [ ] Invoice notifications work
- [ ] Order notifications work
- [ ] Ticket notifications work
- [ ] Priority colors display correctly
- [ ] Channel routing functions
- [ ] Error handling works
