# WHMCS Notification Provider DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/notification-module/
├── Provider.php          # Main notification provider
├── lib/
│   └── ApiClient.php    # API client skeleton
├── logo.png             # 80x80px logo
├── whmcs.json           # Metadata
└── DEVKIT.md            # This file
```

## Template

```php
<?php
/**
 * WHMCS Notification Provider: {provider}
 * Notification Provider Template
 */

namespace WHMCS\Module\Notification\{Provider};

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

class Provider implements NotificationModuleInterface {
    use DescriptionTrait;

    /**
     * Module configuration settings
     * Displayed when configuring the module in WHMCS Admin
     */
    public static function moduleConfiguration(): array {
        return [
            [
                'Name' => 'api_key',
                'Type' => 'password',
                'FriendlyName' => 'API Key',
                'Description' => 'Your provider API key',
            ],
            [
                'Name' => 'api_secret',
                'Type' => 'password',
                'FriendlyName' => 'API Secret',
                'Description' => 'Your provider API secret',
            ],
            [
                'Name' => 'sender_id',
                'Type' => 'text',
                'FriendlyName' => 'Sender ID',
                'Description' => 'Sender identifier for messages',
            ],
            [
                'Name' => 'test_mode',
                'Type' => 'yesno',
                'FriendlyName' => 'Test Mode',
                'Description' => 'Enable test mode for sandbox',
            ],
        ];
    }

    /**
     * Settings for each notification rule
     * Configured per notification channel/rule
     */
    public function notificationSettings(): array {
        return [
            [
                'Name' => 'channel_id',
                'Type' => 'text',
                'FriendlyName' => 'Channel ID',
                'Description' => 'The channel or recipient ID',
            ],
            [
                'Name' => 'template_id',
                'Type' => 'text',
                'FriendlyName' => 'Template ID',
                'Description' => 'Message template ID from provider',
            ],
            [
                'Name' => 'priority',
                'Type' => 'dropdown',
                'Options' => 'low,medium,high,urgent',
                'FriendlyName' => 'Priority',
                'Description' => 'Message priority level',
            ],
        ];
    }

    /**
     * Test connection to the notification provider
     * MUST throw Exception on failure
     */
    public function testConnection(): void {
        $settings = self::getRuntimeSettings();

        try {
            $api = new ApiClient($settings);
            $result = $api->ping();

            if (!$result['success']) {
                throw new \Exception($result['message'] ?? 'Connection failed');
            }
        } catch (\Exception $e) {
            throw new \Exception('Connection test failed: ' . $e->getMessage());
        }
    }

    /**
     * Send the actual notification
     * MUST throw Exception on failure
     */
    public function send(NotificationInterface $notification, array $settings): void {
        $api = new ApiClient($settings);

        $title = $notification->getTitle();
        $message = $notification->getMessage();
        $url = $notification->getUrl();
        $client = $notification->getClient();

        // Build notification payload
        $payload = $this->buildPayload($notification, $settings);

        try {
            $result = $api->send($payload);

            if (!$result['success']) {
                throw new \Exception($result['message'] ?? 'Failed to send notification');
            }
        } catch (\Exception $e) {
            throw new \Exception('Failed to send: ' . $e->getMessage());
        }
    }

    /**
     * Build notification payload from WHMCS notification
     */
    private function buildPayload(NotificationInterface $notification, array $settings): array {
        $title = $notification->getTitle();
        $message = $notification->getMessage();

        // Get attributes for rich notifications
        $attributes = $notification->getAttributes();

        // Build message based on type
        $fullMessage = $title;
        if ($message) {
            $fullMessage .= "\n\n" . $message;
        }

        // Handle URL if present
        $url = $notification->getUrl();
        if ($url) {
            $fullMessage .= "\n\n" . $url;
        }

        return [
            'channel_id' => $settings['channel_id'] ?? '',
            'title' => $title,
            'message' => $fullMessage,
            'priority' => $settings['priority'] ?? 'medium',
            'metadata' => [
                'type' => $notification->getType(),
                'attributes' => $attributes,
            ],
        ];
    }

    /**
     * Get dynamic field configuration for notification rules
     * Optional - use if you need dynamic settings per notification type
     */
    public function getDynamicField(string $fieldName, array $settings): array {
        return match ($fieldName) {
            'template' => [
                'Type' => 'dropdown',
                'Options' => $this->getAvailableTemplates($settings),
                'FriendlyName' => 'Message Template',
            ],
            default => [],
        };
    }

    /**
     * Get available message templates from provider
     */
    private function getAvailableTemplates(array $settings): string {
        try {
            $api = new ApiClient($settings);
            $templates = $api->getTemplates();

            return implode(',', $templates);
        } catch (\Exception $e) {
            return 'default';
        }
    }
}
```

## API Client Skeleton

```php
<?php
namespace WHMCS\Module\Notification\{Provider};

class ApiClient {
    private string $apiUrl;
    private string $apiKey;
    private string $apiSecret;
    private string $senderId;
    private bool $testMode;

    public function __construct(array $settings) {
        $this->apiKey = $settings['api_key'] ?? '';
        $this->apiSecret = $settings['api_secret'] ?? '';
        $this->senderId = $settings['sender_id'] ?? '';
        $this->testMode = ($settings['test_mode'] ?? '') === 'on';
        $this->apiUrl = $this->testMode
            ? 'https://sandbox.api.provider.com'
            : 'https://api.provider.com';
    }

    public function request(string $endpoint, array $data = [], string $method = 'POST'): array {
        $ch = curl_init();

        $headers = [
            'Content-Type: application/json',
            'Authorization: Bearer ' . $this->apiKey,
        ];

        curl_setopt_array($ch, [
            CURLOPT_URL => $this->apiUrl . $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => $headers,
        ]);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        $result = json_decode($response, true);

        if ($httpCode >= 400) {
            throw new \Exception($result['message'] ?? 'API Error: HTTP ' . $httpCode);
        }

        return $result;
    }

    public function ping(): array {
        return $this->request('/ping', [], 'POST');
    }

    public function send(array $payload): array {
        $data = array_merge($payload, [
            'sender' => $this->senderId,
        ]);

        return $this->request('/messages/send', $data);
    }

    public function getTemplates(): array {
        $result = $this->request('/templates', [], 'GET');

        return $result['templates'] ?? [];
    }

    public function getBalance(): float {
        $result = $this->request('/account/balance', [], 'GET');

        return $result['balance'] ?? 0;
    }
}
```

## whmcs.json Metadata

```json
{
    "schema": "1.0",
    "name": "{Provider}",
    "description": "{Provider} notification integration for WHMCS",
    "version": "1.0.0",
    "author": {
        "name": "{Author}",
        "email": "{email@example.com}",
        "homepage": "https://example.com"
    },
    "requires": {
        "php": ">=8.1"
    },
    "whmcs": {
        "min_version": "8.0"
    }
}
```

## Notification Types Reference

```php
// WHMCS notification types you can receive:
$notification->getType();           // 'TicketNotification', 'InvoiceNotification', etc.
$notification->getTitle();           // Title of the notification
$notification->getMessage();        // Message body
$notification->getUrl();            // Action URL
$notification->getClient();         // WHMCS\Client object
$notification->getAttributes();     // Type-specific attributes

// Example attributes for InvoiceNotification:
$attributes['invoice_id'];          // Invoice ID
$attributes['invoice_number'];      // Invoice number
$attributes['amount'];              // Invoice amount
$attributes['date_created'];        // Creation date
$attributes['due_date'];            // Due date

// Example attributes for TicketNotification:
$attributes['ticket_id'];           // Ticket ID
$attributes['ticket_number'];       // Ticket number
$attributes['subject'];             // Ticket subject
$attributes['priority'];            // Priority level

// Example attributes for ServiceNotification:
$attributes['service_id'];           // Service ID
$attributes['service_name'];        // Service name
$attributes['product_name'];        // Product name
$attributes['status'];              // Service status
```

## Checklist

```
Pre-Dev:
□ Get notification provider API documentation
□ Get sandbox/test credentials
□ Understand API authentication method
□ Identify payload structure requirements

Development:
□ Create Provider class with use DescriptionTrait (REQUIRED)
□ Implement moduleConfiguration() → Return settings array
□ Implement notificationSettings() → Return per-rule settings
□ Implement testConnection() → throw Exception on failure
□ Implement send() → throw Exception on failure
□ Create ApiClient class
□ Create logo.png (80x80px)
□ Create whmcs.json metadata

Security:
□ Never log full API keys
□ Validate all inputs
□ Timeout API calls (max 30s)
□ Handle rate limiting gracefully

Testing:
□ Test connection with valid credentials
□ Test connection with invalid credentials → should throw Exception
□ Test sending notification
□ Test sending with invalid recipient → should throw Exception
□ Verify logo displays correctly
□ Test in WHMCS notification rules
```
