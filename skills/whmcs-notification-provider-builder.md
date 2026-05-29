# WHMCS Notification Provider Builder

## Concept

Notification providers allow WHMCS to send notifications through custom channels like Slack, Discord, Teams, Pushbullet, etc. They extend the notification system to support any communication platform.

## File Structure

```
/modules/notifications/
├── yourprovider/
│   ├── yourprovider.php    # Main notification provider
│   └── logo.png            # Provider logo
```

## Core Notification Provider

```php
<?php
/**
 * Notification Provider: Your Provider
 * Version: 1.0.0
 * Description: Send notifications via Your Provider
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Notifications\NotificationProvider;
use WHMCS\Notifications\NotificationBody;

class YourProviderNotificationProvider extends NotificationProvider
{
    public function defineSettings()
    {
        return [
            'api_key' => [
                'FriendlyName' => 'API Key',
                'Type' => 'password',
                'Description' => 'Enter your API key from the dashboard',
            ],
            'webhook_url' => [
                'FriendlyName' => 'Webhook URL',
                'Type' => 'text',
                'Description' => 'The webhook URL for sending messages',
            ],
            'channel' => [
                'FriendlyName' => 'Default Channel',
                'Type' => 'text',
                'Description' => 'Default channel to post to',
            ],
        ];
    }
    
    public function send(NotificationBody $body, array $settings)
    {
        $webhookUrl = $settings['webhook_url'];
        $apiKey = $settings['api_key'];
        
        $payload = $this->buildPayload($body);
        
        return $this->makeRequest($webhookUrl, $payload, $apiKey);
    }
    
    protected function buildPayload(NotificationBody $body)
    {
        $payload = [
            'username' => 'WHMCS Notifications',
            'avatar_url' => 'https://yourprovider.com/logo.png',
            'embeds' => [],
        ];
        
        // Parse notification highlight
        $highlight = $body->getHighlight();
        
        $embed = [
            'title' => $body->getSubject(),
            'description' => $body->getMessage(),
            'color' => $this->getColor($body->getPriority()),
            'fields' => [],
            'footer' => [
                'text' => 'WHMCS',
                'icon_url' => 'https://yourprovider.com/whmcs-icon.png',
            ],
            'timestamp' => date('c'),
        ];
        
        // Add action URL if available
        if ($actionUrl = $body->getActionUrl()) {
            $embed['url'] = $actionUrl;
        }
        
        // Add fields from notification
        foreach ($body->getFields() as $field) {
            $embed['fields'][] = [
                'name' => $field['label'],
                'value' => $field['value'],
                'inline' => $field['inline'] ?? false,
            ];
        }
        
        $payload['embeds'][] = $embed;
        
        return $payload;
    }
    
    protected function getColor(string $priority)
    {
        switch ($priority) {
            case 'high':
                return 15158332; // Red
            case 'medium':
                return 15105570; // Orange
            case 'low':
            default:
                return 3447003; // Blue
        }
    }
    
    protected function makeRequest(string $url, array $payload, string $apiKey)
    {
        $ch = curl_init($url);
        
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'Authorization: Bearer ' . $apiKey,
            ],
            CURLOPT_TIMEOUT => 30,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode >= 400) {
            throw new \Exception('Failed to send notification: HTTP ' . $httpCode);
        }
        
        return true;
    }
}
```

## Service Provider Registration

```php
<?php
// In your module file, register the notification provider

use WHMCS\Module\Notification\YourProviderNotificationProvider;

return [
    'provider' => YourProviderNotificationProvider::class,
];
```

## Advanced Provider with Custom Fields

```php
<?php
class SlackNotificationProvider extends NotificationProvider
{
    public function defineSettings()
    {
        return [
            'webhook_url' => [
                'FriendlyName' => 'Webhook URL',
                'Type' => 'text',
                'Required' => true,
                'Description' => 'Your Slack webhook URL (https://hooks.slack.com/...)',
            ],
            'channel' => [
                'FriendlyName' => 'Channel Override',
                'Type' => 'text',
                'Description' => 'Override the default channel (leave blank for webhook channel)',
            ],
            'bot_name' => [
                'FriendlyName' => 'Bot Name',
                'Type' => 'text',
                'Default' => 'WHMCS Bot',
            ],
            'icon' => [
                'FriendlyName' => 'Icon Emoji',
                'Type' => 'text',
                'Default' => ':robot_face:',
                'Description' => 'Slack emoji for bot icon',
            ],
            'include_details' => [
                'FriendlyName' => 'Include Details',
                'Type' => 'yesno',
                'Description' => 'Include additional details in the notification',
            ],
        ];
    }
    
    public function send(NotificationBody $body, array $settings)
    {
        $webhookUrl = $settings['webhook_url'];
        
        // Override channel if specified
        if (!empty($settings['channel'])) {
            $webhookUrl = str_replace('/services/', '/' . $settings['channel'] . '/', $webhookUrl);
        }
        
        $payload = [
            'username' => $settings['bot_name'] ?? 'WHMCS',
            'icon_emoji' => $settings['icon'] ?? ':robot_face:',
            'text' => $body->getSubject(),
            'attachments' => [
                [
                    'text' => $body->getMessage(),
                    'color' => $this->getStatusColor($body),
                    'fields' => [],
                ],
            ],
        ];
        
        // Add detail fields
        if (!empty($settings['include_details'])) {
            foreach ($body->getFields() as $field) {
                $payload['attachments'][0]['fields'][] = [
                    'title' => $field['label'],
                    'value' => $field['value'],
                    'short' => $field['short'] ?? true,
                ];
            }
        }
        
        // Add action button if available
        if ($actionUrl = $body->getActionUrl()) {
            $payload['attachments'][0]['actions'] = [
                [
                    'type' => 'button',
                    'text' => 'View Details',
                    'url' => $actionUrl,
                ],
            ];
        }
        
        return $this->makeRequest($webhookUrl, $payload);
    }
    
    protected function getStatusColor(NotificationBody $body)
    {
        $highlight = $body->getHighlight();
        
        switch ($highlight) {
            case 'important':
            case 'danger':
                return '#FF0000';
            case 'warning':
                return '#FFA500';
            case 'success':
                return '#36A64F';
            default:
                return '#439FE0';
        }
    }
}
```

## Testing the Provider

```php
// Add a test method to verify configuration
public function testConnection(array $settings)
{
    try {
        $testPayload = [
            'text' => 'Test notification from WHMCS',
            'attachments' => [
                [
                    'text' => 'If you see this, your notification provider is working correctly.',
                ],
            ],
        ];
        
        $this->makeRequest($settings['webhook_url'], $testPayload);
        
        return [
            'success' => true,
            'message' => 'Test notification sent successfully',
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'message' => 'Failed to send test notification: ' . $e->getMessage(),
        ];
    }
}
```

## Usage in Hooks

```php
// Sending notification from a hook
add_hook('ServiceCreated', 1, function($params) {
    $notification = DI::make('Notification');
    
    $notification->send(
        'service_created',
        [
            'client' => $params['client'],
            'service' => $params['service'],
        ],
        [
            'provider' => 'yourprovider',
        ]
    );
});
```

## Step-by-Step Implementation

1. Create notification provider directory
2. Create provider class extending NotificationProvider
3. Implement defineSettings method
4. Implement send method with payload building
5. Add provider logo
6. Test notification delivery
7. Verify in WHMCS notification settings

## Implementation Checklist

- [ ] Create provider directory in modules/notifications
- [ ] Implement provider class extending NotificationProvider
- [ ] Implement defineSettings method
- [ ] Implement send method
- [ ] Build notification payload
- [ ] Add provider logo
- [ ] Handle errors properly
- [ ] Add test connection method
- [ ] Register provider in service provider format
- [ ] Test in WHMCS admin notification settings