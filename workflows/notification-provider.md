# WHMCS Notification Provider Workflow
# Version: 1.0 | Created: 2026-05-28

---

## Overview

This workflow guides the creation of WHMCS notification provider modules for SMS, push notifications, and chat integrations.

## Prerequisites

1. Read `.agents-whmcs/CLAUDE.md` (Technical Reference)
2. Read `Core_exapm_whmcs/sample-notification-module/`
3. Read `.agents/docs/WHMCS_ADDON_NOTIFICATION.md`

---

## Module Structure

```
modules/notifications/{Provider}/
├── {Provider}.php            ← Main notification file
├── logo.png                  ← 80x80px logo
└── whmcs.json               ← Module metadata
```

---

## Step-by-Step Development

### Step 1: Create Main Notification File

Create `{Provider}.php`:

```php
<?php
/**
 * {Provider} Notification Provider for WHMCS
 * Version: 1.0.0
 * Author: HiTechCloud
 */

namespace WHMCS\Module\Notification;

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use Illuminate\Support\Facades\Http;
use Psr\Http\Message\ServerRequestInterface;

if (!defined("WHMCS")) {
    die("Direct access denied");
}

/**
 * {Provider} Notification Provider
 */
class {Provider} implements NotificationModuleInterface
{
    use DescriptionTrait; // REQUIRED - provides getDisplayName(), getLogoFileName()

    /**
     * Module Configuration (one-time settings)
     */
    public static function moduleConfiguration(): array
    {
        return [
            [
                'Name' => 'api_key',
                'Type' => 'password',
                'FriendlyName' => 'API Key',
                'Description' => 'Your {Provider} API key',
            ],
            [
                'Name' => 'api_secret',
                'Type' => 'password',
                'FriendlyName' => 'API Secret',
                'Description' => 'Your {Provider} API secret',
            ],
            [
                'Name' => 'webhook_url',
                'Type' => 'text',
                'FriendlyName' => 'Webhook URL',
                'Description' => 'Optional: URL to receive delivery reports',
            ],
        ];
    }

    /**
     * Test Connection
     * IMPORTANT: Throw Exception on failure, do NOT return false
     */
    public function testConnection(): void
    {
        $apiKey = $this->getSetting('api_key');
        $apiSecret = $this->getSetting('api_secret');

        if (empty($apiKey) || empty($apiSecret)) {
            throw new \Exception('API credentials not configured');
        }

        try {
            // Test connection to provider
            $response = $this->sendApiRequest('GET', '/account/verify');

            if (!isset($response['success']) || !$response['success']) {
                throw new \Exception('Invalid API credentials');
            }
        } catch (\Exception $e) {
            throw new \Exception('Connection failed: ' . $e->getMessage());
        }
    }

    /**
     * Notification Rule Settings (per-notification configuration)
     */
    public function notificationSettings(): array
    {
        return [
            [
                'Name' => 'recipient_type',
                'Type' => 'dropdown',
                'FriendlyName' => 'Recipient Type',
                'Options' => [
                    'client_mobile' => 'Client Mobile Number',
                    'admin_mobile' => 'Admin Mobile Number',
                    'static' => 'Static Number',
                ],
                'Default' => 'client_mobile',
            ],
            [
                'Name' => 'static_recipient',
                'Type' => 'text',
                'FriendlyName' => 'Static Recipient',
                'Description' => 'Phone number when recipient_type is Static',
            ],
            [
                'Name' => 'template',
                'Type' => 'textarea',
                'FriendlyName' => 'Message Template',
                'Description' => 'Available: {title}, {message}, {url}',
            ],
        ];
    }

    /**
     * Send Notification
     * IMPORTANT: Throw Exception on failure, do NOT return false
     */
    public function send(NotificationInterface $notification, array $settings): void
    {
        $apiKey = $this->getSetting('api_key');
        $apiSecret = $this->getSetting('api_secret');

        // Get recipient
        $recipient = $this->resolveRecipient($settings);

        if (empty($recipient)) {
            throw new \Exception('No recipient configured');
        }

        // Build message
        $title = $notification->getTitle();
        $message = $notification->getMessage();
        $url = $notification->getUrl();

        // Apply template if configured
        $template = $settings['template'] ?? '{title}: {message}';
        $body = str_replace(
            ['{title}', '{message}', '{url}'],
            [$title, $message, $url],
            $template
        );

        // Send notification
        try {
            $result = $this->sendApiRequest('POST', '/messages/send', [
                'recipient' => $recipient,
                'message' => $body,
            ]);

            if (!isset($result['success']) || !$result['success']) {
                throw new \Exception($result['error'] ?? 'Failed to send');
            }
        } catch (\Exception $e) {
            throw new \Exception('Send failed: ' . $e->getMessage());
        }
    }

    /**
     * Dynamic Field (optional - for dynamic settings)
     */
    public function getDynamicField(string $fieldName, array $settings): array
    {
        // Return dynamic field configuration based on field name
        return [];
    }

    /**
     * Display Name (from DescriptionTrait)
     */
    public function getDisplayName(): string
    {
        return '{Provider} Notifications';
    }

    /**
     * Logo File Name (from DescriptionTrait)
     * Logo must be 80x80px PNG
     */
    public function getLogoFileName(): string
    {
        return 'logo.png';
    }

    // Helper methods

    private function getSetting(string $name): string
    {
        // Get setting value from WHMCS
        $config = \Config::getInstance();
        return $config->get('modNotifications_{Provider}_' . $name) ?? '';
    }

    private function resolveRecipient(array $settings): string
    {
        $type = $settings['recipient_type'] ?? 'client_mobile';

        switch ($type) {
            case 'client_mobile':
                // Get from client data (needs to be passed via notification rule)
                return $settings['client_mobile'] ?? '';

            case 'admin_mobile':
                // Get from admin data
                return $settings['admin_mobile'] ?? '';

            case 'static':
                return $settings['static_recipient'] ?? '';

            default:
                return '';
        }
    }

    private function sendApiRequest(string $method, string $endpoint, array $data = []): array
    {
        $apiKey = $this->getSetting('api_key');
        $apiSecret = $this->getSetting('api_secret');

        $ch = curl_init();
        $url = 'https://api.provider.com' . $endpoint;

        $headers = [
            'Authorization: Bearer ' . $apiKey,
            'X-Api-Secret: ' . $apiSecret,
            'Content-Type: application/json',
        ];

        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_SSL_VERIFYPEER => true,
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
            throw new \Exception('API Error: ' . $error);
        }

        return json_decode($response, true) ?? [];
    }
}
```

### Step 2: Create whmcs.json

Create `whmcs.json`:

```json
{
    "schema": "1.0",
    "type": "notification-provider",
    "name": {
        "en": "{Provider} Notifications"
    },
    "description": {
        "en": "Send notifications via {Provider} SMS and messaging"
    },
    "version": "1.0.0",
    "authors": [
        {
            "name": "HiTechCloud",
            "homepage": "https://hitechcloud.vn"
        }
    ],
    "categories": ["notifications", "sms"]
}
```

### Step 3: Create Logo

Create `logo.png` (80x80px PNG, transparent background recommended)

---

## Notification Types in WHMCS

| Event | Title | Message | URL |
|-------|-------|---------|-----|
| New Order | "New Order #{order_id}" | "Order for {product_name}" | Order link |
| Invoice Created | "Invoice #{invoice_id}" | "Invoice for {amount}" | Invoice link |
| Invoice Paid | "Payment Received" | "Invoice #{invoice_id} paid" | Invoice link |
| Service Created | "Service Activated" | "{product_name} is now active" | Service link |
| Service Suspended | "Service Suspended" | "{product_name} has been suspended" | Service link |
| Ticket Opened | "Support Ticket #{ticket_id}" | "{subject}" | Ticket link |
| Password Reset | "Password Reset Request" | "Click to reset password" | Reset link |

---

## Checklist

- [ ] `use DescriptionTrait` - REQUIRED
- [ ] `implements NotificationModuleInterface`
- [ ] `moduleConfiguration()` - returns array of setting fields
- [ ] `testConnection()` - throw Exception on failure
- [ ] `notificationSettings()` - returns per-rule settings
- [ ] `send()` - throw Exception on failure
- [ ] `getDisplayName()` - returns provider name
- [ ] `getLogoFileName()` - returns 'logo.png'
- [ ] Logo file (80x80px)
- [ ] whmcs.json metadata

---

## Common Issues

| Issue | Solution |
|-------|----------|
| Module not showing in admin | Check `use DescriptionTrait` is present |
| testConnection returns false but should throw | Use `throw new \Exception()` not `return false` |
| Notifications not sending | Check `send()` uses `throw new \Exception()` |
| Logo not displaying | Verify logo.png is 80x80px PNG |

---

## Vietnamese Notification Providers

For Vietnamese providers like Zalo ZBS:

```php
// Zalo ZBS pattern
public function send(NotificationInterface $notification, array $settings): void
{
    $accessToken = $this->getAccessToken();

    $payload = [
        'phone' => $this->resolveRecipient($settings),
        'template_id' => $settings['template_id'],
        'template_data' => [
            'customer_name' => $notification->getTitle(),
            'message' => $notification->getMessage(),
        ],
    ];

    $response = $this->callZaloApi('/zsender/phone/template', $accessToken, $payload);

    if (!$response['success']) {
        throw new \Exception('Zalo send failed: ' . $response['message']);
    }
}
```

---

Last updated: 2026-05-28