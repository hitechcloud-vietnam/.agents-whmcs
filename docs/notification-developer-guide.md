# WHMCS Notification Developer Guide

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

Notification modules allow WHMCS to send alerts and updates through various channels like SMS, push notifications, Slack, Discord, and custom services. This guide covers the complete development process for custom notification providers.

---

## Module Structure

```
modules/notifications/
  your_notification/
    your_notification.php     # Main module file
    templates/
      default.tpl            # Default notification template
      alert.tpl              # Alert template
    assets/
      icon.png               # Notification provider icon
```

## Main Module File

```php
<?php
/**
 * Notification Module Definition
 * 
 * @package WHMCS
 * @subpackage NotificationModule
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Define module meta data
 * 
 * @return array Module metadata
 */
function your_notification_MetaData()
{
    return [
        'DisplayName'          => 'Your Notification Provider',
        'APIVersion'          => 1.0,
        'Description'         => 'Send notifications via Your Provider',
        'Languages'           => ['english'],
        'Requires'            => [
            'Php' => '7.4',
        ],
    ];
}

/**
 * Get configuration fields
 * 
 * @return array Configuration fields
 */
function your_notification_config()
{
    return [
        'FriendlyName' => [
            'Type'    => 'System',
            'Value'   => 'Your Notification Provider',
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type'         => 'password',
            'Size'         => '50',
            'Description'  => 'Your API key from the notification provider',
        ],
        'api_secret' => [
            'FriendlyName' => 'API Secret',
            'Type'         => 'password',
            'Size'         => '50',
        ],
        'default_channel' => [
            'FriendlyName' => 'Default Channel',
            'Type'         => 'text',
            'Size'         => '50',
            'Description'  => 'Default channel or recipient',
        ],
        'test_mode' => [
            'FriendlyName' => 'Test Mode',
            'Type'         => 'yesno',
            'Description'  => 'Enable to send to test channel only',
        ],
        'rate_limit' => [
            'FriendlyName' => 'Rate Limit (per hour)',
            'Type'         => 'text',
            'Size'         => '10',
            'Default'      => '100',
        ],
    ];
}
```

## Notification Sending

```php
/**
 * Send notification
 * 
 * @param array $params Notification parameters
 * @param array $notification Notification data
 * @return array Send result
 */
function your_notification_send($params, $notification)
{
    $apiKey    = $params['api_key'];
    $apiSecret = $params['api_secret'];
    $testMode  = $params['test_mode'];
    
    // Determine recipient
    $recipients = your_notification_getRecipients($params, $notification);
    
    // Build message payload
    $payload = your_notification_buildPayload($notification, $recipients, $testMode);
    
    // Send via API
    $endpoint = 'https://api.yournprovider.com/messages/send';
    
    $ch = curl_init($endpoint);
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => json_encode($payload),
        CURLOPT_HTTPHEADER     => [
            'Authorization: Bearer ' . $apiKey,
            'Content-Type: application/json',
        ],
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT        => 30,
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    // Log the notification attempt
    logModuleCall(
        'your_notification',
        'send',
        $payload,
        $response,
        null,
        [$apiKey] // Mask sensitive data
    );
    
    if ($httpCode >= 200 && $httpCode < 300) {
        return [
            'status'  => 'success',
            'details' => $result,
        ];
    }
    
    return [
        'status' => 'failed',
        'message' => $result['error']['message'] ?? 'Send failed',
        'details' => $result,
    ];
}

/**
 * Get recipients based on notification type
 * 
 * @param array $params Module parameters
 * @param array $notification Notification data
 * @return array Recipients
 */
function your_notification_getRecipients($params, $notification)
{
    $recipients = [];
    
    // Admin notifications
    if (!empty($notification['admin_output'])) {
        $admins = your_notification_getAdminRecipients($params);
        $recipients = array_merge($recipients, $admins);
    }
    
    // User-specific notifications
    if (!empty($notification['user_id'])) {
        $userChannel = your_notification_getUserChannel($notification['user_id']);
        if ($userChannel) {
            $recipients[] = $userChannel;
        }
    }
    
    // Default channel
    if (empty($recipients) && !empty($params['default_channel'])) {
        $recipients[] = $params['default_channel'];
    }
    
    return array_unique($recipients);
}

/**
 * Get admin recipients
 * 
 * @param array $params Module parameters
 * @return array Admin recipients
 */
function your_notification_getAdminRecipients($params)
{
    $query = "SELECT DISTINCT `email` FROM `tbladmins` 
              WHERE `disabled` = 0 AND `email` != ''";
    $result = full_query($query);
    
    $admins = [];
    while ($row = mysql_fetch_array($result, MYSQLI_ASSOC)) {
        $admins[] = $row['email'];
    }
    
    return $admins;
}

/**
 * Get user notification channel
 * 
 * @param int $userId User ID
 * @return array|null User channel info
 */
function your_notification_getUserChannel($userId)
{
    $result = select_query('custom_user_channels', '*', ['user_id' => $userId]);
    $data = mysql_fetch_array($result, MYSQLI_ASSOC);
    
    if ($data) {
        return [
            'type'    => $data['channel_type'],
            'address' => $data['channel_address'],
            'user_id' => $userId,
        ];
    }
    
    return null;
}

/**
 * Build notification payload
 * 
 * @param array $notification Notification data
 * @param array $recipients Recipients
 * @param bool $testMode Test mode flag
 * @return array API payload
 */
function your_notification_buildPayload($notification, $recipients, $testMode)
{
    $payload = [
        'recipients' => $recipients,
        'type'       => $notification['type'] ?? 'info',
        'message'    => your_notification_formatMessage($notification),
        'metadata'   => [
            'whmcs_event'     => $notification['event'],
            'whmcs_timestamp' => date('c'),
            'notification_id' => $notification['id'] ?? null,
        ],
    ];
    
    // Add attachments if present
    if (!empty($notification['attachments'])) {
        $payload['attachments'] = $notification['attachments'];
    }
    
    // Test mode marker
    if ($testMode) {
        $payload['test_mode'] = true;
    }
    
    return $payload;
}

/**
 * Format notification message
 * 
 * @param array $notification Notification data
 * @return string Formatted message
 */
function your_notification_formatMessage($notification)
{
    $type = $notification['type'] ?? 'info';
    $title = $notification['title'] ?? 'WHMCS Notification';
    $message = $notification['message'] ?? '';
    
    switch ($type) {
        case 'error':
            return "{$title}: {$message}";
            
        case 'warning':
            return "{$title}: {$message}";
            
        case 'success':
            return "{$title}: {$message}";
            
        default:
            return "{$title}\n{$message}";
    }
}
```

## Contact Discovery

```php
/**
 * Discover contact information
 * 
 * @param array $params Module parameters
 * @param string $accountId Account identifier
 * @param array $userData User information
 * @return array Contact discovery result
 */
function your_notification_discoverContacts($params, $accountId, $userData)
{
    $contacts = [];
    
    // Primary email
    if (!empty($userData['email'])) {
        $contacts[] = [
            'type'    => 'email',
            'address' => $userData['email'],
            'label'   => 'Primary Email',
            'primary' => true,
        ];
    }
    
    // Phone number
    if (!empty($userData['phonenumber'])) {
        $contacts[] = [
            'type'    => 'phone',
            'address' => $userData['phonenumber'],
            'label'   => 'Phone Number',
            'primary' => false,
        ];
    }
    
    // Mobile number (if different)
    if (!empty($userData['mobilenumber'])) {
        $contacts[] = [
            'type'    => 'sms',
            'address' => $userData['mobilenumber'],
            'label'   => 'Mobile Number',
            'primary' => false,
        ];
    }
    
    return [
        'success'  => true,
        'contacts' => $contacts,
        'account'  => $accountId,
    ];
}

/**
 * Get available contact types
 * 
 * @return array Available contact types
 */
function your_notification_getContactTypes()
{
    return [
        'email',
        'sms',
        'push',
        'slack',
        'discord',
        'telegram',
    ];
}
```

## Webhook Configuration

```php
/**
 * Define webhook settings
 * 
 * @return array Webhook configuration
 */
function your_notification_webhookSettings()
{
    return [
        'webhook_enabled' => [
            'FriendlyName' => 'Enable Webhooks',
            'Type'         => 'yesno',
            'Description'  => 'Receive incoming webhooks for two-way communication',
        ],
        'webhook_secret' => [
            'FriendlyName' => 'Webhook Secret',
            'Type'         => 'password',
            'Description'  => 'Secret for verifying incoming webhook signatures',
        ],
        'webhook_url' => [
            'FriendlyName' => 'Webhook URL',
            'Type'         => 'System',
            'Description'  => 'Use this URL to configure incoming webhooks',
        ],
    ];
}

/**
 * Process incoming webhook
 * 
 * @param array $params Module parameters
 * @param array $payload Webhook payload
 * @return array Processing result
 */
function your_notification_processWebhook($params, $payload)
{
    // Verify webhook signature
    $signature = $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? '';
    $secret = $params['webhook_secret'];
    
    $expectedSig = hash_hmac('sha256', json_encode($payload), $secret);
    
    if (!hash_equals($expectedSig, $signature)) {
        return [
            'status'  => 'failed',
            'message' => 'Invalid signature',
        ];
    }
    
    // Process the webhook
    $event = $payload['event'] ?? 'unknown';
    $data = $payload['data'] ?? [];
    
    switch ($event) {
        case 'message_delivered':
            your_notification_handleDelivery($data);
            break;
            
        case 'message_failed':
            your_notification_handleFailure($data);
            break;
            
        case 'opt_out':
            your_notification_handleOptOut($data);
            break;
            
        default:
            logModuleCall('your_notification', 'webhook', $payload, 'Unknown event');
    }
    
    return [
        'status' => 'success',
    ];
}
```

## Template Management

```php
/**
 * Get notification template
 * 
 * @param string $templateName Template name
 * @param array $variables Template variables
 * @return string Rendered template
 */
function your_notification_getTemplate($templateName, $variables)
{
    $templatePath = __DIR__ . '/templates/' . $templateName . '.tpl';
    
    if (!file_exists($templatePath)) {
        $templatePath = __DIR__ . '/templates/default.tpl';
    }
    
    $template = file_get_contents($templatePath);
    
    // Replace variables
    foreach ($variables as $key => $value) {
        $template = str_replace('{$' . $key . '}', $value, $template);
    }
    
    return $template;
}

/**
 * List available notification types
 * 
 * @param array $params Module parameters
 * @return array Available notification types
 */
function your_notification_listNotificationTypes($params)
{
    return [
        'invoice_created',
        'invoice_paid',
        'invoice_overdue',
        'ticket_opened',
        'ticket_reply',
        'ticket_close',
        'order_new',
        'order_accepted',
        'order_cancelled',
        'domain_renewal',
        'password_reset',
        'custom_alert',
    ];
}
```

## Installation Checklist

1. Create notification module directory
2. Implement module metadata function
3. Define configuration fields
4. Implement send function
5. Add webhook processing (if supported)
6. Create notification templates
7. Add contact discovery
8. Test notification flow
9. Verify rate limiting
10. Document configuration options

## Testing Checklist

- [ ] Test send without recipients
- [ ] Test single recipient
- [ ] Test multiple recipients
- [ ] Verify rate limiting
- [ ] Test template rendering
- [ ] Verify error handling
- [ ] Test webhook signature verification
- [ ] Test contact discovery

## Rate Limiting Implementation

```php
/**
 * Check and update rate limit
 * 
 * @param array $params Module parameters
 * @return bool True if allowed
 */
function your_notification_checkRateLimit($params)
{
    $limit = (int) ($params['rate_limit'] ?? 100);
    $window = 3600; // 1 hour window
    
    $now = time();
    $windowStart = $now - $window;
    
    // Count recent notifications
    $query = "SELECT COUNT(*) as `count` FROM `mod_notification_log` 
              WHERE `created_at` > FROM_UNIXTIME({$windowStart})";
    $result = full_query($query);
    $row = mysql_fetch_array($result);
    
    if ($row['count'] >= $limit) {
        return false;
    }
    
    return true;
}
```

---

## Related Skills and Workflows

- `module-logging-guide` - Logging notification attempts
- `module-error-handling-guide` - Error handling patterns
- `module-security-standards` - Webhook security
- `webhook-events-reference` - Event handling
- `email-template-variables` - Email template variables
