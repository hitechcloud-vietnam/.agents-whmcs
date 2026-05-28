# WHMCS Notification Provider DevKit

A comprehensive notification provider module for WHMCS that integrates with external notification services (Slack, Discord, custom webhooks, etc.).

## Features

- Multiple notification channel support
- Customizable message templates
- Priority-based delivery
- Queue-based processing
- Rich formatting (Slack, Discord, Email)
- Webhook integration
- Comprehensive error handling

## Installation

1. Copy provider files to:
   ```
   modules/notifications/{provider}/
   ```

2. Activate the provider in WHMCS Admin:
   - Go to Configuration > System Settings > Notification Methods
   - Find your provider and click Activate

3. Configure the provider:
   - Go to Configuration > System Settings > Notification Methods
   - Click Configure for your provider
   - Enter API credentials and settings

## Configuration

### Module Settings (Global)

| Setting | Description |
|---------|-------------|
| API Key | Your notification service API key |
| API Secret | Your notification service API secret |
| Webhook URL | Custom webhook endpoint |
| Default Channel | Default notification channel |
| From Name | Sender name |
| Enabled Events | Which events trigger notifications |

### Notification Settings (Per Rule)

| Setting | Description |
|---------|-------------|
| Channel ID | Target channel for this rule |
| Template | Message template (default, minimal, detailed) |
| Priority | low, normal, high, urgent |
| Include Attachments | Include file attachments |

## Usage

### Notification Types

The provider supports all WHMCS notification types:

- **Invoice**: Invoice created, paid, overdue, cancelled
- **Support**: Ticket opened, replied, closed
- **Service**: Service created, suspended, terminated, upgraded
- **Domain**: Domain registered, renewed, transferred, expired
- **Order**: Order placed, accepted, fulfilled, cancelled
- **General**: Custom notifications

### Message Templates

#### Default Template
```
Title: Notification title
Body: Notification message
Fields: Up to 5 key-value pairs
```

#### Minimal Template
```
Title: Short title
Body: Message truncated to 200 chars
```

#### Detailed Template
```
Title: Full title
Body: Complete message
Fields: All available fields
Actions: Clickable action buttons
```

### Slack Formatting

When using Slack template, messages are formatted with:
- Bold title
- Plain text body
- Color-coded attachment (based on type)
- Key-value field table

### Discord Formatting

Discord messages include:
- Rich embeds
- Color-coded borders
- Field tables
- Timestamp

## Webhook Integration

The provider can send notifications to custom webhook endpoints:

```php
// Configure webhook URL in module settings
$settings = [
    'webhook_url' => 'https://your-webhook.com/endpoint',
];

// Notifications are sent as POST requests with JSON body
$payload = [
    'channel' => 'channel-id',
    'message' => [
        'title' => 'Invoice Paid',
        'body' => 'Invoice #123 has been paid',
        'fields' => [...],
    ],
    'priority' => 'normal',
];
```

## API Reference

### Provider Class Methods

```php
// Module configuration
public static function moduleConfiguration(): array { }

// Per-rule notification settings
public function notificationSettings(): array { }

// Test connection
public function testConnection(): void { }

// Send notification
public function send(NotificationInterface $notification, array $settings): void { }
```

### NotificationHandler Class

```php
// Create notification
$notification = NotificationHandler::createNotification('invoice', $data);

// Queue notification
$handler->queue('provider_name', $data, $settings);

// Process queue
$processed = $handler->processQueue();

// Get statistics
$stats = $handler->getStats();
```

### MessageBuilder Class

```php
// Build message
$message = (new MessageBuilder('slack'))
    ->set('title', 'Invoice Paid')
    ->set('message', 'Invoice #123 has been paid')
    ->set('client_name', 'John Doe')
    ->set('amount', '$99.99')
    ->build();
```

## Hook Integration

Send notifications on WHMCS events:

```php
// Invoice paid notification
add_hook('InvoicePaid', 1, function($vars) {
    $notification = new \WHMCS\Notification\Notification();
    $notification->setTitle('Invoice Paid')
        ->setMessage("Invoice #{$vars['invoiceid']} has been paid")
        ->setType('invoice')
        ->setAttributes([
            'invoice_id' => $vars['invoiceid'],
            'amount' => $vars['total'],
        ]);
    
    // WHMCS handles sending via configured providers
});

// Support ticket notification
add_hook('TicketOpen', 1, function($vars) {
    $notification = new \WHMCS\Notification\Notification();
    $notification->setTitle('Support Ticket Opened')
        ->setMessage($vars['subject'])
        ->setType('support')
        ->setAttributes([
            'ticket_id' => $vars['ticketid'],
        ]);
});
```

## Error Handling

The provider throws exceptions on failure (not returning false):

```php
// Connection failed
throw new \Exception('Failed to connect to notification service');

// Invalid credentials
throw new \Exception('Invalid API credentials');

// Send failed
throw new \Exception('Failed to send notification');
```

## Logging

All notification activity is logged in WHMCS admin:

1. Go to Configuration > System Logs
2. Filter by notification provider
3. View detailed logs including:
   - Timestamp
   - Notification type
   - Recipient
   - Status
   - Error messages

## File Structure

```
whmcs-notification-provider/
├── provider.php              # Main provider class
├── lib/
│   ├── NotificationHandler.php # Notification handling
│   └── MessageBuilder.php     # Template builder
└── templates/
    └── notification-config.tpl # Admin configuration
```

## Requirements

- WHMCS 8.0+ (Notification system)
- PHP 7.4+
- cURL extension

## Supported Services

- Slack (via webhook or API)
- Discord (via webhook)
- Microsoft Teams (via webhook)
- Custom REST APIs
- Generic webhooks

## Support

For issues and feature requests, please contact the developer.