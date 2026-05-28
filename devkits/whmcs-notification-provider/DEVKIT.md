# WHMCS Notification Provider DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-notification-provider/
├── provider.php             # Notification provider
├── lib/
│   ├── NotificationHandler.php # Notification handling
│   └── MessageBuilder.php      # Message template builder
└── templates/
    └── notification-config.tpl # Admin configuration
```

## Main Notification Provider

```php
<?php
/**
 * WHMCS Notification Provider: {Provider}
 * DevKit Template
 * 
 * Provides custom notification delivery
 * Installation: Copy to modules/notifications/{provider}/
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

namespace WHMCS\Module\Notification\{Provider};

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

/**
 * Notification Provider Class
 */
class Provider implements NotificationModuleInterface {
    
    use DescriptionTrait;
    
    /**
     * Module configuration
     */
    public static function moduleConfiguration(): array {
        return [
            [
                'Name' => 'api_key',
                'Type' => 'password',
                'FriendlyName' => 'API Key',
                'Description' => 'Your notification service API key',
            ],
            [
                'Name' => 'api_secret',
                'Type' => 'password',
                'FriendlyName' => 'API Secret',
                'Description' => 'Your notification service API secret',
            ],
            [
                'Name' => 'webhook_url',
                'Type' => 'text',
                'FriendlyName' => 'Webhook URL',
                'Description' => 'Custom webhook endpoint (optional)',
            ],
            [
                'Name' => 'channel',
                'Type' => 'text',
                'FriendlyName' => 'Default Channel',
                'Description' => 'Default notification channel (e.g., channel ID)',
            ],
            [
                'Name' => 'from_name',
                'Type' => 'text',
                'FriendlyName' => 'From Name',
                'Description' => 'Sender name for notifications',
            ],
            [
                'Name' => 'enabled_events',
                'Type' => 'dropdown',
                'FriendlyName' => 'Enabled Events',
                'Options' => 'all,invoice,support,service,domain,order',
                'Description' => 'Which events trigger notifications',
            ],
        ];
    }
    
    /**
     * Notification settings (per-rule)
     */
    public function notificationSettings(): array {
        return [
            [
                'Name' => 'channel_id',
                'Type' => 'text',
                'FriendlyName' => 'Channel ID',
                'Description' => 'Target channel for this notification rule',
            ],
            [
                'Name' => 'template',
                'Type' => 'dropdown',
                'FriendlyName' => 'Message Template',
                'Options' => 'default,detailed,minimal',
            ],
            [
                'Name' => 'priority',
                'Type' => 'dropdown',
                'FriendlyName' => 'Priority',
                'Options' => 'low,normal,high,urgent',
            ],
            [
                'Name' => 'include_attachment',
                'Type' => 'yesno',
                'FriendlyName' => 'Include Attachments',
                'Description' => 'Include file attachments in notifications',
            ],
        ];
    }
    
    /**
     * Test connection
     */
    public function testConnection(): void {
        $apiKey = $this->getModulename()::getMapping()['api_key'];
        
        if (empty($apiKey)) {
            throw new \Exception('API key is required');
        }
        
        // Test API connection
        $response = $this->apiCall('GET', '/test', []);
        
        if (!isset($response['success']) || !$response['success']) {
            throw new \Exception('Failed to connect to notification service');
        }
    }
    
    /**
     * Send notification
     */
    public function send(NotificationInterface $notification, array $settings): void {
        $message = $this->buildMessage($notification, $settings);
        
        // Determine channel
        $channelId = $settings['channel_id'] ?? $settings['channel'] ?? null;
        
        if (empty($channelId)) {
            throw new \Exception('Channel ID is required');
        }
        
        // Send via API
        $response = $this->apiCall('POST', '/messages/send', [
            'channel' => $channelId,
            'message' => $message,
            'priority' => $settings['priority'] ?? 'normal',
            'metadata' => [
                'notification_type' => $notification->getType(),
                'notification_id' => $notification->getId(),
                'timestamp' => date('c'),
            ],
        ]);
        
        if (!isset($response['success']) || !$response['success']) {
            throw new \Exception(
                $response['error'] ?? 'Failed to send notification'
            );
        }
    }
    
    /**
     * Build message from notification
     */
    private function buildMessage(NotificationInterface $notification, array $settings): array {
        $template = $settings['template'] ?? 'default';
        
        $title = $notification->getTitle();
        $body = $notification->getMessage();
        
        // Build message based on template
        switch ($template) {
            case 'minimal':
                return [
                    'title' => $title,
                    'body' => substr($body, 0, 200),
                ];
                
            case 'detailed':
                return [
                    'title' => $title,
                    'body' => $body,
                    'fields' => $notification->toArray(),
                    'actions' => $this->buildActions($notification),
                ];
                
            default: // 'default'
                return [
                    'title' => $title,
                    'body' => $body,
                    'fields' => array_slice($notification->toArray(), 0, 5),
                ];
        }
    }
    
    /**
     * Build action buttons
     */
    private function buildActions(NotificationInterface $notification): array {
        $actions = [];
        
        // Add view action if available
        if ($notification->getUrl()) {
            $actions[] = [
                'text' => 'View Details',
                'url' => $notification->getUrl(),
                'style' => 'primary',
            ];
        }
        
        return $actions;
    }
    
    /**
     * Make API call
     */
    private function apiCall(string $method, string $endpoint, array $data = []): array {
        $apiKey = $this->getModulename()::getMapping()['api_key'];
        $apiSecret = $this->getModulename()::getMapping()['api_secret'];
        
        $ch = curl_init();
        
        $url = rtrim($this->getSetting('webhook_url') ?: 'https://api.notification-service.com', '/') . $endpoint;
        
        $headers = [
            'Authorization: Bearer ' . $apiKey,
            'Content-Type: application/json',
            'X-API-Secret: ' . $apiSecret,
        ];
        
        $options = [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => $headers,
        ];
        
        if ($method !== 'GET' && !empty($data)) {
            $options[CURLOPT_POST] = true;
            $options[CURLOPT_POSTFIELDS] = json_encode($data);
        }
        
        curl_setopt_array($ch, $options);
        
        $response = curl_exec($ch);
        $error = curl_error($ch);
        curl_close($ch);
        
        if ($response === false) {
            throw new \Exception('API Error: ' . $error);
        }
        
        $result = json_decode($response, true);
        
        if (json_last_error() !== JSON_ERROR_NONE) {
            throw new \Exception('Invalid API response');
        }
        
        return $result;
    }
    
    /**
     * Get setting value
     */
    private function getSetting(string $name): ?string {
        $module = $this->getModulename();
        return $module::getMapping()[$name] ?? null;
    }
}
```

## Notification Handler

```php
<?php
/**
 * Notification Handler
 */

namespace NotificationHandler;

use WHMCS\Database\Capsule;
use WHMCS\Notification\NotificationInterface;

class NotificationHandler {
    
    private array $channels = [];
    private array $templates = [];
    
    /**
     * Create notification
     */
    public static function createNotification(string $type, array $data): NotificationInterface {
        $class = "\\WHMCS\\Notification\\Contracts\\Notification";
        
        // Build notification based on type
        switch ($type) {
            case 'invoice':
                return new InvoiceNotification($data);
            case 'support':
                return new SupportNotification($data);
            case 'service':
                return new ServiceNotification($data);
            case 'domain':
                return new DomainNotification($data);
            case 'order':
                return new OrderNotification($data);
            default:
                return new GenericNotification($data);
        }
    }
    
    /**
     * Queue notification
     */
    public function queue(string $provider, array $data, array $settings = []): int {
        return Capsule::table('mod_{module}_notification_queue')->insertGetId([
            'provider' => $provider,
            'notification_type' => $data['type'] ?? 'generic',
            'notification_data' => json_encode($data),
            'settings' => json_encode($settings),
            'status' => 'pending',
            'attempts' => 0,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    /**
     * Process queue
     */
    public function processQueue(): int {
        $pending = Capsule::table('mod_{module}_notification_queue')
            ->where('status', 'pending')
            ->where('attempts', '<', 5)
            ->limit(50)
            ->get();
        
        $processed = 0;
        
        foreach ($pending as $item) {
            try {
                $data = json_decode($item->notification_data, true);
                $settings = json_decode($item->settings, true);
                
                // Create notification
                $notification = self::createNotification($item->notification_type, $data);
                
                // Send via provider (requires provider instance)
                // This would be handled by the notification module
                
                Capsule::table('mod_{module}_notification_queue')
                    ->where('id', $item->id)
                    ->update(['status' => 'completed']);
                
                $processed++;
                
            } catch (\Exception $e) {
                Capsule::table('mod_{module}_notification_queue')
                    ->where('id', $item->id)
                    ->update([
                        'attempts' => $item->attempts + 1,
                        'last_error' => $e->getMessage(),
                    ]);
                
                if ($item->attempts >= 4) {
                    Capsule::table('mod_{module}_notification_queue')
                        ->where('id', $item->id)
                        ->update(['status' => 'failed']);
                }
            }
        }
        
        return $processed;
    }
    
    /**
     * Get notification statistics
     */
    public function getStats(): array {
        return [
            'pending' => Capsule::table('mod_{module}_notification_queue')
                ->where('status', 'pending')
                ->count(),
            
            'completed' => Capsule::table('mod_{module}_notification_queue')
                ->where('status', 'completed')
                ->count(),
            
            'failed' => Capsule::table('mod_{module}_notification_queue')
                ->where('status', 'failed')
                ->count(),
            
            'today' => Capsule::table('mod_{module}_notification_queue')
                ->whereDate('created_at', date('Y-m-d'))
                ->count(),
        ];
    }
}
```

## Message Builder

```php
<?php
/**
 * Message Template Builder
 */

namespace NotificationHandler;

class MessageBuilder {
    
    private string $template;
    private array $variables = [];
    
    public function __construct(string $template = 'default') {
        $this->template = $template;
    }
    
    /**
     * Set template variable
     */
    public function set(string $key, $value): self {
        $this->variables[$key] = $value;
        return $this;
    }
    
    /**
     * Set multiple variables
     */
    public function setMany(array $variables): self {
        $this->variables = array_merge($this->variables, $variables);
        return $this;
    }
    
    /**
     * Build message
     */
    public function build(): array {
        return match ($this->template) {
            'minimal' => $this->buildMinimal(),
            'detailed' => $this->buildDetailed(),
            'slack' => $this->buildSlack(),
            'discord' => $this->buildDiscord(),
            'email' => $this->buildEmail(),
            default => $this->buildDefault(),
        };
    }
    
    /**
     * Build default template
     */
    private function buildDefault(): array {
        return [
            'title' => $this->variables['title'] ?? 'Notification',
            'body' => $this->variables['message'] ?? '',
            'fields' => array_filter([
                'client' => $this->variables['client_name'] ?? null,
                'service' => $this->variables['service_name'] ?? null,
                'amount' => $this->variables['amount'] ?? null,
            ]),
        ];
    }
    
    /**
     * Build minimal template
     */
    private function buildMinimal(): array {
        return [
            'title' => $this->variables['title'] ?? 'Notification',
            'body' => substr($this->variables['message'] ?? '', 0, 200),
        ];
    }
    
    /**
     * Build detailed template
     */
    private function buildDetailed(): array {
        $fields = [];
        
        foreach ($this->variables as $key => $value) {
            if (!in_array($key, ['title', 'message', 'url']) && $value) {
                $fields[ucfirst(str_replace('_', ' ', $key))] = $value;
            }
        }
        
        return [
            'title' => $this->variables['title'] ?? 'Notification',
            'body' => $this->variables['message'] ?? '',
            'fields' => $fields,
            'url' => $this->variables['url'] ?? null,
            'timestamp' => date('c'),
        ];
    }
    
    /**
     * Build Slack-formatted message
     */
    private function buildSlack(): array {
        $text = "*" . ($this->variables['title'] ?? 'Notification') . "*\n";
        $text .= ($this->variables['message'] ?? '') . "\n\n";
        
        foreach (['client_name', 'service_name', 'amount', 'invoice_id'] as $key) {
            if (!empty($this->variables[$key])) {
                $label = ucfirst(str_replace('_', ' ', $key));
                $text .= "_{$label}:_ {$this->variables[$key]}\n";
            }
        }
        
        return [
            'text' => $text,
            'attachments' => [
                [
                    'color' => $this->getSlackColor(),
                    'fields' => array_map(function($key, $value) {
                        return [
                            'title' => ucfirst(str_replace('_', ' ', $key)),
                            'value' => $value,
                            'short' => true,
                        ];
                    }, array_keys($this->variables), array_values($this->variables)),
                ],
            ],
        ];
    }
    
    /**
     * Build Discord-formatted message
     */
    private function buildDiscord(): array {
        return [
            'content' => '',
            'embeds' => [
                [
                    'title' => $this->variables['title'] ?? 'Notification',
                    'description' => $this->variables['message'] ?? '',
                    'color' => $this->getDiscordColor(),
                    'fields' => array_map(function($key, $value) {
                        return [
                            'name' => ucfirst(str_replace('_', ' ', $key)),
                            'value' => (string) $value,
                            'inline' => true,
                        ];
                    }, array_keys($this->variables), array_values($this->variables)),
                    'timestamp' => date('c'),
                ],
            ],
        ];
    }
    
    /**
     * Build email template
     */
    private function buildEmail(): array {
        $html = '<h2>' . ($this->variables['title'] ?? 'Notification') . '</h2>';
        $html .= '<p>' . nl2br($this->variables['message'] ?? '') . '</p>';
        $html .= '<hr>';
        $html .= '<table>';
        
        foreach ($this->variables as $key => $value) {
            if (!in_array($key, ['title', 'message', 'url']) && $value) {
                $html .= '<tr><td><strong>' . ucfirst(str_replace('_', ' ', $key)) . ':</strong></td>';
                $html .= '<td>' . $value . '</td></tr>';
            }
        }
        
        $html .= '</table>';
        
        if (!empty($this->variables['url'])) {
            $html .= '<p><a href="' . $this->variables['url'] . '">View Details</a></p>';
        }
        
        return [
            'subject' => $this->variables['title'] ?? 'Notification',
            'html' => $html,
            'text' => strip_tags($this->variables['message'] ?? ''),
        ];
    }
    
    /**
     * Get Slack color based on type
     */
    private function getSlackColor(): string {
        $type = $this->variables['type'] ?? 'info';
        
        return match ($type) {
            'success', 'completed' => '#36a64f',
            'warning', 'pending' => '#ff9800',
            'error', 'failed' => '#dc3545',
            default => '#17a2b8',
        };
    }
    
    /**
     * Get Discord color based on type
     */
    private function getDiscordColor(): int {
        $type = $this->variables['type'] ?? 'info';
        
        return match ($type) {
            'success', 'completed' => 0x36a64f,
            'warning', 'pending' => 0xff9800,
            'error', 'failed' => 0xdc3545,
            default => 0x17a2b8,
        };
    }
}
```

## Hook Integration

```php
<?php
/**
 * Notification Provider Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Send notification on invoice paid
add_hook('InvoicePaid', 1, function($vars) {
    $notification = new \WHMCS\Notification\Notification();
    $notification->setTitle('Invoice Paid')
        ->setMessage("Invoice #{$vars['invoiceid']} has been paid")
        ->setType('invoice')
        ->setUrl("viewinvoice.php?id={$vars['invoiceid']}")
        ->setAttributes([
            'invoice_id' => $vars['invoiceid'],
            'amount' => $vars['total'],
            'client_id' => $vars['userid'],
        ]);
    
    // Send via provider
    // This is handled by WHMCS notification system
});

// Send notification on ticket open
add_hook('TicketOpen', 1, function($vars) {
    $notification = new \WHMCS\Notification\Notification();
    $notification->setTitle('Support Ticket Opened')
        ->setMessage($vars['subject'])
        ->setType('support')
        ->setUrl("supporttickets.php?action=viewticket&id={$vars['ticketid']}")
        ->setAttributes([
            'ticket_id' => $vars['ticketid'],
            'subject' => $vars['subject'],
        ]);
});
```

## Admin Configuration Template

```smarty
<div class="notification-provider">
    <h2>{Provider} Notification Settings</h2>
    
    <form method="post" action="{$smarty.server.PHP_SELF}">
        <input type="hidden" name="module" value="notifications">
        <input type="hidden" name="action" value="module_settings">
        <input type="hidden" name="module_name" value="{provider}">
        <input type="hidden" name="csrf_token" value="{$csrf_token}">
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">API Configuration</h3>
            </div>
            <div class="panel-body">
                <div class="form-group">
                    <label>API Key</label>
                    <input type="password" name="api_key" class="form-control" 
                           value="{$modulename::getMapping()['api_key']}">
                </div>
                
                <div class="form-group">
                    <label>API Secret</label>
                    <input type="password" name="api_secret" class="form-control"
                           value="{$modulename::getMapping()['api_secret']}">
                </div>
                
                <div class="form-group">
                    <label>Webhook URL</label>
                    <input type="url" name="webhook_url" class="form-control"
                           value="{$modulename::getMapping()['webhook_url']}"
                           placeholder="https://api.example.com">
                </div>
            </div>
        </div>
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Notification Settings</h3>
            </div>
            <div class="panel-body">
                <div class="form-group">
                    <label>Default Channel</label>
                    <input type="text" name="channel" class="form-control"
                           value="{$modulename::getMapping()['channel']}"
                           placeholder="channel-id or channel-name">
                </div>
                
                <div class="form-group">
                    <label>From Name</label>
                    <input type="text" name="from_name" class="form-control"
                           value="{$modulename::getMapping()['from_name']}"
                           placeholder="WHMCS Notifications">
                </div>
                
                <div class="form-group">
                    <label>Enabled Events</label>
                    <select name="enabled_events" class="form-control">
                        <option value="all" {if $modulename::getMapping()['enabled_events'] == 'all'}selected{/if}>
                            All Events
                        </option>
                        <option value="invoice" {if $modulename::getMapping()['enabled_events'] == 'invoice'}selected{/if}>
                            Invoice Only
                        </option>
                        <option value="support" {if $modulename::getMapping()['enabled_events'] == 'support'}selected{/if}>
                            Support Only
                        </option>
                        <option value="service" {if $modulename::getMapping()['enabled_events'] == 'service'}selected{/if}>
                            Service Only
                        </option>
                    </select>
                </div>
            </div>
        </div>
        
        <button type="submit" class="btn btn-primary">
            <i class="fa fa-save"></i> Save Configuration
        </button>
        
        <button type="button" class="btn btn-info" onclick="testConnection()">
            <i class="fa fa-plug"></i> Test Connection
        </button>
    </form>
</div>

<script>
function testConnection() {
    $.post('notifications.php', {
        action: 'test_connection',
        module: '{provider}'
    }, function(response) {
        if (response.success) {
            alert('Connection successful!');
        } else {
            alert('Connection failed: ' + response.error);
        }
    }, 'json');
}
</script>
```

## Checklist

```
Pre-Dev:
□ Define notification service API
□ Plan message templates
□ Identify priority levels
□ Design notification settings

Development:
□ Create provider class implementing NotificationModuleInterface
□ Add DescriptionTrait
□ Implement moduleConfiguration()
□ Implement notificationSettings()
□ Implement testConnection()
□ Implement send()
□ Create NotificationHandler class
□ Create MessageBuilder class
□ Add hook integrations
□ Build admin configuration UI

Testing:
□ Test connection
□ Test notification sending
□ Verify template rendering
□ Test priority handling
□ Verify error handling
```