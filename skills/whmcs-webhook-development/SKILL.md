# WHMCS Webhook Development Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for developing outgoing webhooks in WHMCS modules for real-time event notifications to external systems.

## When to Use

- Building webhook event systems
- Sending notifications to external services
- Integrating with third-party systems via webhooks
- Event-driven architecture

## Webhook Development Patterns

### 1. Webhook Dispatcher Class

```php
<?php
namespace WHMCS\Webhook;

class WebhookDispatcher {
    private array $config;
    private int $maxRetries = 3;
    private int $timeout = 30;

    public function __construct() {
        $this->loadConfiguration();
    }

    public function dispatch(string $event, array $payload, array $recipients = []): DispatchResult {
        $eventConfig = $this->getEventConfig($event);
        if (!$eventConfig || !$eventConfig['enabled']) {
            return new DispatchResult(['status' => 'skipped', 'reason' => 'Event not enabled']);
        }

        $recipients = empty($recipients) ? $this->getRecipients($event) : $recipients;

        if (empty($recipients)) {
            return new DispatchResult(['status' => 'skipped', 'reason' => 'No recipients']);
        }

        $result = new DispatchResult();
        $enrichedPayload = $this->enrichPayload($event, $payload);

        foreach ($recipients as $recipient) {
            $deliveryResult = $this->deliverToRecipient($recipient, $event, $enrichedPayload);
            $result->addDelivery($recipient['url'], $deliveryResult);
        }

        return $result;
    }

    private function enrichPayload(string $event, array $payload): array {
        $enriched = [
            'event' => $event,
            'timestamp' => date('c'),
            'whmcs_url' => rtrim($this->config['system_url'] ?? '', '/'),
            'data' => $payload,
        ];

        // Add client data if available
        if (isset($payload['user_id'])) {
            $client = Capsule::table('tblclients')->find($payload['user_id']);
            if ($client) {
                $enriched['client'] = [
                    'id' => $client->id,
                    'email' => $client->email,
                    'full_name' => trim($client->firstname . ' ' . $client->lastname),
                ];
            }
        }

        // Add service data if available
        if (isset($payload['service_id'])) {
            $service = Capsule::table('tblhosting')->find($payload['service_id']);
            if ($service) {
                $enriched['service'] = [
                    'id' => $service->id,
                    'domain' => $service->domain,
                    'status' => $service->domainstatus,
                ];
            }
        }

        return $enriched;
    }

    private function deliverToRecipient(array $recipient, string $event, array $payload): array {
        $url = $recipient['url'];
        $secret = $recipient['secret'] ?? '';
        $headers = $this->buildHeaders($event, $secret);
        $startTime = microtime(true);

        try {
            $ch = curl_init();
            curl_setopt_array($ch, [
                CURLOPT_URL => $url,
                CURLOPT_RETURNTRANSFER => true,
                CURLOPT_TIMEOUT => $this->timeout,
                CURLOPT_POST => true,
                CURLOPT_POSTFIELDS => json_encode($payload),
                CURLOPT_HTTPHEADER => $headers,
                CURLOPT_SSL_VERIFYPEER => true,
                CURLOPT_SSL_VERIFYHOST => 2,
            ]);

            $response = curl_exec($ch);
            $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            $error = curl_error($ch);
            curl_close($ch);

            $duration = microtime(true) - $startTime;

            if ($httpCode >= 200 && $httpCode < 300) {
                return [
                    'status' => 'success',
                    'http_code' => $httpCode,
                    'response' => $response,
                    'duration_ms' => round($duration * 1000),
                ];
            }

            return [
                'status' => 'failed',
                'http_code' => $httpCode,
                'error' => $error ?: "HTTP $httpCode",
                'duration_ms' => round($duration * 1000),
            ];

        } catch (\Exception $e) {
            return [
                'status' => 'error',
                'error' => $e->getMessage(),
                'duration_ms' => round((microtime(true) - $startTime) * 1000),
            ];
        }
    }

    private function buildHeaders(string $event, string $secret): array {
        $timestamp = time();
        $signature = $secret ? $this->generateSignature($event, $timestamp, $secret) : '';

        $headers = [
            'Content-Type: application/json',
            'Accept: application/json',
            'X-Webhook-Event: ' . $event,
            'X-Webhook-Timestamp: ' . $timestamp,
        ];

        if ($signature) {
            $headers[] = 'X-Webhook-Signature: ' . $signature;
        }

        return $headers;
    }

    private function generateSignature(string $event, int $timestamp, string $secret): string {
        $payload = json_encode(['event' => $event, 'timestamp' => $timestamp]);
        return hash_hmac('sha256', $payload, $secret);
    }

    private function getEventConfig(string $event): ?array {
        $config = Capsule::table('mod_webhook_events')
            ->where('event', $event)
            ->first();

        return $config ? (array) $config : null;
    }

    private function getRecipients(string $event): array {
        return Capsule::table('mod_webhook_recipients')
            ->where('event', $event)
            ->where('active', 1)
            ->get()
            ->toArray();
    }

    private function loadConfiguration(): void {
        $this->config = [
            'system_url' => Capsule::table('tblconfiguration')->where('setting', 'SystemURL')->first()->value ?? '',
        ];
    }
}
```

### 2. Webhook Result Class

```php
<?php
namespace WHMCS\Webhook;

class DispatchResult {
    private string $status;
    private string $reason = '';
    private array $deliveries = [];

    public function __construct(array $config = []) {
        $this->status = $config['status'] ?? 'success';
        $this->reason = $config['reason'] ?? '';
    }

    public function addDelivery(string $url, array $result): void {
        $this->deliveries[$url] = $result;

        if ($result['status'] !== 'success' && $this->status === 'success') {
            $this->status = 'partial';
        }
    }

    public function getStatus(): string {
        if ($this->status !== 'partial' && $this->status !== 'skipped' && !empty($this->deliveries)) {
            $allSuccess = true;
            foreach ($this->deliveries as $delivery) {
                if ($delivery['status'] !== 'success') {
                    $allSuccess = false;
                    break;
                }
            }
            $this->status = $allSuccess ? 'success' : 'partial';
        }
        return $this->status;
    }

    public function getDeliveries(): array {
        return $this->deliveries;
    }

    public function getSuccessCount(): int {
        return count(array_filter($this->deliveries, fn($d) => $d['status'] === 'success'));
    }

    public function getFailedCount(): int {
        return count(array_filter($this->deliveries, fn($d) => $d['status'] !== 'success'));
    }

    public function toArray(): array {
        return [
            'status' => $this->getStatus(),
            'reason' => $this->reason,
            'deliveries' => $this->deliveries,
            'success_count' => $this->getSuccessCount(),
            'failed_count' => $this->getFailedCount(),
        ];
    }
}
```

### 3. Webhook Management

```php
<?php
namespace WHMCS\Webhook;

class WebhookManager {
    public function registerEvent(string $event, string $label, string $description = ''): int {
        return Capsule::table('mod_webhook_events')->insertGetId([
            'event' => $event,
            'label' => $label,
            'description' => $description,
            'enabled' => 1,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function addRecipient(string $event, string $url, string $secret = ''): int {
        return Capsule::table('mod_webhook_recipients')->insertGetId([
            'event' => $event,
            'url' => $url,
            'secret' => $secret,
            'active' => 1,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function toggleEvent(string $event, bool $enabled): void {
        Capsule::table('mod_webhook_events')
            ->where('event', $event)
            ->update(['enabled' => $enabled ? 1 : 0]);
    }

    public function removeRecipient(int $id): void {
        Capsule::table('mod_webhook_recipients')
            ->where('id', $id)
            ->delete();
    }

    public function getWebhookLogs(int $limit = 100): array {
        return Capsule::table('mod_webhook_logs')
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    public function retryFailedWebhook(int $logId): bool {
        $log = Capsule::table('mod_webhook_logs')->find($logId);
        if (!$log || $log->status !== 'failed') {
            return false;
        }

        $dispatcher = new WebhookDispatcher();
        $result = $dispatcher->dispatch($log->event, json_decode($log->payload, true), [
            ['url' => $log->recipient_url, 'secret' => $log->secret],
        ]);

        // Update log
        Capsule::table('mod_webhook_logs')
            ->where('id', $logId)
            ->update([
                'status' => $result->getStatus(),
                'response' => json_encode($result->getDeliveries()),
                'retry_count' => $log->retry_count + 1,
                'last_attempt' => date('Y-m-d H:i:s'),
            ]);

        return $result->getStatus() === 'success';
    }
}
```

### 4. Hook Integration

```php
<?php
// hooks.php - Register webhook events and dispatch
add_hook('AfterModuleCreate', 1, function($vars) {
    $dispatcher = new WebhookDispatcher();
    $dispatcher->dispatch('module.created', [
        'user_id' => $vars['userid'],
        'service_id' => $vars['serviceid'],
        'server_id' => $vars['serverid'],
        'package_id' => $vars['packageid'],
    ]);
});

add_hook('InvoicePaid', 1, function($vars) {
    $dispatcher = new WebhookDispatcher();
    $dispatcher->dispatch('invoice.paid', [
        'invoice_id' => $vars['invoice_id'],
        'amount' => $vars['amount'],
        'payment_method' => $vars['payment_method'],
    ]);
});

add_hook('ClientAdd', 1, function($vars) {
    $dispatcher = new WebhookDispatcher();
    $dispatcher->dispatch('client.created', [
        'user_id' => $vars['id'],
        'email' => $vars['email'],
    ]);
});

add_hook('TicketOpen', 1, function($vars) {
    $dispatcher = new WebhookDispatcher();
    $dispatcher->dispatch('ticket.opened', [
        'ticket_id' => $vars['ticket_id'],
        'subject' => $vars['subject'],
        'priority' => $vars['priority'],
    ]);
});

// Register default events on module activation
add_hook('{Module}_activate', 1, function($vars) {
    $manager = new WebhookManager();

    // Register default events
    $events = [
        'client.created' => 'Client Created',
        'client.updated' => 'Client Updated',
        'module.created' => 'Service Created',
        'module.suspended' => 'Service Suspended',
        'module.terminated' => 'Service Terminated',
        'invoice.paid' => 'Invoice Paid',
        'invoice.created' => 'Invoice Created',
        'domain.registered' => 'Domain Registered',
        'domain.renewed' => 'Domain Renewed',
        'ticket.opened' => 'Support Ticket Opened',
    ];

    foreach ($events as $event => $label) {
        $manager->registerEvent($event, $label);
    }
});
```

### 5. Database Schema

```php
<?php
function createWebhookTables(): void {
    Capsule::schema()->create('mod_webhook_events', function($t) {
        $t->increments('id');
        $t->string('event')->unique();
        $t->string('label');
        $t->text('description')->nullable();
        $t->boolean('enabled')->default(1);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_webhook_recipients', function($t) {
        $t->increments('id');
        $t->string('event');
        $t->string('url');
        $t->text('secret')->nullable();
        $t->boolean('active')->default(1);
        $t->timestamps();

        $t->foreign('event')->references('event')->on('mod_webhook_events')->onDelete('cascade');
    });

    Capsule::schema()->create('mod_webhook_logs', function($t) {
        $t->bigIncrements('id');
        $t->string('event');
        $t->string('recipient_url');
        $t->text('payload'); // JSON
        $t->string('status'); // success, failed, pending
        $t->text('response')->nullable();
        $t->integer('retry_count')->unsigned()->default(0);
        $t->timestamp('created_at');
        $t->timestamp('last_attempt')->nullable();
        $t->integer('duration_ms')->unsigned()->nullable();

        $t->index(['event', 'created_at']);
        $t->index(['status']);
    });
}
```

### 6. Admin Interface

```php
<?php
// modules/addons/{module}/admin/webhooks.php
if (!defined("WHMCS")) { die("Direct access denied"); }

use WHMCS\Database\Capsule;
use WHMCS\Webhook\WebhookManager;

// Handle form submissions
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.admin.default');
    $manager = new WebhookManager();

    if ($_POST['action'] === 'add_recipient') {
        $manager->addRecipient(
            $_POST['event'],
            $_POST['url'],
            $_POST['secret'] ?? ''
        );
        flash('success', 'Webhook recipient added');
    } elseif ($_POST['action'] === 'toggle_event') {
        $manager->toggleEvent($_POST['event'], isset($_POST['enabled']));
        flash('success', 'Event settings updated');
    } elseif ($_POST['action'] === 'remove_recipient') {
        $manager->removeRecipient((int) $_POST['id']);
        flash('success', 'Recipient removed');
    } elseif ($_POST['action'] === 'retry') {
        $manager->retryFailedWebhook((int) $_POST['id']);
        flash('success', 'Webhook retry initiated');
    }

    redirect('addonmodules.php?module={module}&action=webhooks');
}

// Load data
$events = Capsule::table('mod_webhook_events')->get();
$recipients = Capsule::table('mod_webhook_recipients')->get();
$logs = Capsule::table('mod_webhook_logs')->orderBy('created_at', 'desc')->limit(50)->get();

// Display views/templates/admin/webhooks.tpl
```

## Checklist

- [ ] Webhook dispatcher class
- [ ] Event registration system
- [ ] Recipient management
- [ ] Signature generation and verification
- [ ] HTTPS delivery with proper headers
- [ ] Error handling and retry logic
- [ ] Webhook logging
- [ ] Admin configuration interface
- [ ] Hook integration for WHMCS events
- [ ] Manual retry functionality

---

**Related Skills:**
- whmcs-webhook-handler
- whmcs-webhook-integration
- whmcs-hooks-development
- whmcs-api-integration
