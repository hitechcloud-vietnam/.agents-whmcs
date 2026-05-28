# WHMCS Webhook Listener DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-webhook-listener/
├── webhook-listener.php    # Main webhook listener
├── lib/
│   ├── EventProcessor.php  # Event processing logic
│   ├── WebhookValidator.php # Request validation
│   └── EventHandlers.php   # Event handler implementations
├── templates/
│   └── webhook_config.tpl  # Admin configuration
└── webhook-admin.php       # Admin interface
```

## Main Webhook Listener

```php
<?php
/**
 * WHMCS Webhook Listener
 * DevKit Template
 * 
 * Handles incoming webhooks from WHMCS
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {module}_config(): array {
    return [
        'name' => '{Webhook Listener}',
        'description' => 'Incoming webhook listener with event handling',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Activate
 */
function {module}_activate(): array {
    Capsule::schema()->create('mod_{module}_webhooks', function($t) {
        $t->increments('id');
        $t->string('event_type');
        $t->string('webhook_url');
        $t->text('headers');
        $t->boolean('is_active');
        $t->integer('retry_count');
        $t->integer('max_retries')->default(3);
        $t->timestamp('last_triggered')->nullable();
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_webhook_logs', function($t) {
        $t->increments('id');
        $t->string('event_type');
        $t->text('payload');
        $t->text('response');
        $t->integer('status_code');
        $t->boolean('success');
        $t->string('error_message')->nullable();
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_webhook_events', function($t) {
        $t->increments('id');
        $t->string('event_type');
        $t->text('payload');
        $t->string('status')->default('pending');
        $t->integer('attempts')->default(0);
        $t->timestamp('next_attempt')->nullable();
        $t->timestamp('processed_at')->nullable();
        $t->timestamp('created_at');
    });
    
    return ['status' => 'success', 'description' => 'Webhook listener activated'];
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_webhooks');
    Capsule::schema()->dropIfExists('mod_{module}_webhook_logs');
    Capsule::schema()->dropIfExists('mod_{module}_webhook_events');
    
    return ['status' => 'success'];
}

/**
 * Webhook endpoint handler
 */
function {module}_webhook_endpoint(): void {
    // Validate request
    $validator = new WebhookValidator();
    
    if (!$validator->validateRequest()) {
        http_response_code(400);
        echo json_encode(['error' => 'Invalid request']);
        exit;
    }
    
    // Get webhook data
    $payload = file_get_contents('php://input');
    $eventData = json_decode($payload, true);
    
    if (json_last_error() !== JSON_ERROR_NONE) {
        http_response_code(400);
        echo json_encode(['error' => 'Invalid JSON payload']);
        exit;
    }
    
    // Process event
    $processor = new EventProcessor();
    $result = $processor->process($eventData);
    
    if ($result['success']) {
        http_response_code(200);
        echo json_encode(['status' => 'processed', 'event_id' => $result['event_id']]);
    } else {
        http_response_code(500);
        echo json_encode(['error' => $result['error']]);
    }
    exit;
}

/**
 * Output function (Admin Interface)
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
    
    switch ($action) {
        case 'settings':
            {module}_showSettings();
            break;
        case 'logs':
            {module}_viewLogs();
            break;
        case 'webhooks':
            {module}_manageWebhooks();
            break;
        case 'test':
            {module}_testWebhook();
            break;
        default:
            {module}_showDashboard();
    }
}
```

## Webhook Validator

```php
<?php
/**
 * Webhook Request Validator
 */

namespace WebhookListener;

class WebhookValidator {
    
    private string $secretKey;
    private array $allowedIps = [];
    
    public function __construct() {
        $this->secretKey = Capsule::table('mod_{module}_settings')
            ->where('setting', 'webhook_secret')
            ->value('value') ?? '';
        
        $ipList = Capsule::table('mod_{module}_settings')
            ->where('setting', 'allowed_ips')
            ->value('value') ?? '';
        
        $this->allowedIps = $ipList ? array_filter(array_map('trim', explode(',', $ipList))) : [];
    }
    
    /**
     * Validate incoming webhook request
     */
    public function validateRequest(): bool {
        // Check IP whitelist if configured
        if (!empty($this->allowedIps)) {
            $clientIp = $this->getClientIp();
            if (!in_array($clientIp, $this->allowedIps)) {
                logActivity("Webhook rejected: IP {$clientIp} not in whitelist");
                return false;
            }
        }
        
        // Validate signature if configured
        if (!empty($this->secretKey)) {
            $signature = $_SERVER['HTTP_X_WHMCS_SIGNATURE'] ?? $_SERVER['HTTP_X_SIGNATURE'] ?? '';
            
            if (empty($signature)) {
                logActivity("Webhook rejected: Missing signature");
                return false;
            }
            
            $payload = file_get_contents('php://input');
            $expectedSignature = hash_hmac('sha256', $payload, $this->secretKey);
            
            if (!hash_equals($expectedSignature, $signature)) {
                logActivity("Webhook rejected: Invalid signature");
                return false;
            }
        }
        
        // Validate required fields
        $payload = file_get_contents('php://input');
        $data = json_decode($payload, true);
        
        if (!isset($data['event'])) {
            logActivity("Webhook rejected: Missing event type");
            return false;
        }
        
        return true;
    }
    
    /**
     * Generate webhook signature
     */
    public function generateSignature(string $payload): string {
        return hash_hmac('sha256', $payload, $this->secretKey);
    }
    
    /**
     * Get client IP address
     */
    private function getClientIp(): string {
        $ipKeys = ['HTTP_CF_CONNECTING_IP', 'HTTP_X_FORWARDED_FOR', 'REMOTE_ADDR'];
        
        foreach ($ipKeys as $key) {
            if (!empty($_SERVER[$key])) {
                $ip = $_SERVER[$key];
                if (strpos($ip, ',') !== false) {
                    $ip = trim(explode(',', $ip)[0]);
                }
                return $ip;
            }
        }
        
        return $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0';
    }
    
    /**
     * Validate event type
     */
    public function isValidEventType(string $eventType): bool {
        $validEvents = [
            'ClientAdd',
            'ClientEdit',
            'ClientDelete',
            'AfterModuleCreate',
            'AfterModuleSuspend',
            'AfterModuleUnsuspend',
            'AfterModuleTerminate',
            'InvoicePaid',
            'InvoiceCancelled',
            'InvoiceCreated',
            'TicketOpen',
            'TicketReply',
            'TicketClose',
            'AcceptOrder',
            'AfterServiceChangePackage',
            'AfterRegistrarRegistration',
            'AfterRegistrarRenewal',
            'AfterRegistrarTransfer',
            'DailyCronJob',
        ];
        
        return in_array($eventType, $validEvents);
    }
}
```

## Event Processor

```php
<?php
/**
 * Webhook Event Processor
 */

namespace WebhookListener;

use WHMCS\Database\Capsule;

class EventProcessor {
    
    private array $handlers = [];
    private bool $asyncProcessing = false;
    
    public function __construct() {
        $this->loadHandlers();
    }
    
    /**
     * Load registered event handlers
     */
    private function loadHandlers(): void {
        $webhooks = Capsule::table('mod_{module}_webhooks')
            ->where('is_active', 1)
            ->get();
        
        foreach ($webhooks as $webhook) {
            $this->handlers[$webhook->event_type][] = [
                'id' => $webhook->id,
                'url' => $webhook->webhook_url,
                'headers' => json_decode($webhook->headers, true) ?? [],
            ];
        }
        
        // Check if async processing is enabled
        $asyncSetting = Capsule::table('mod_{module}_settings')
            ->where('setting', 'async_processing')
            ->value('value');
        
        $this->asyncProcessing = $asyncSetting === '1';
    }
    
    /**
     * Process incoming event
     */
    public function process(array $eventData): array {
        $eventType = $eventData['event'] ?? 'unknown';
        
        // Store event
        $eventId = Capsule::table('mod_{module}_webhook_events')->insertGetId([
            'event_type' => $eventType,
            'payload' => json_encode($eventData),
            'status' => $this->asyncProcessing ? 'pending' : 'processing',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        if ($this->asyncProcessing) {
            // Queue for async processing
            $this->scheduleAsyncProcessing($eventId);
            return ['success' => true, 'event_id' => $eventId, 'queued' => true];
        }
        
        // Process synchronously
        return $this->processEvent($eventId, $eventData);
    }
    
    /**
     * Process event and trigger webhooks
     */
    private function processEvent(int $eventId, array $eventData): array {
        $eventType = $eventData['event'] ?? 'unknown';
        
        $handlers = $this->handlers[$eventType] ?? [];
        
        if (empty($handlers)) {
            Capsule::table('mod_{module}_webhook_events')
                ->where('id', $eventId)
                ->update([
                    'status' => 'completed',
                    'processed_at' => date('Y-m-d H:i:s'),
                ]);
            return ['success' => true, 'event_id' => $eventId, 'no_handlers' => true];
        }
        
        $success = true;
        $errors = [];
        
        foreach ($handlers as $handler) {
            $result = $this->triggerWebhook($handler, $eventData);
            
            if (!$result['success']) {
                $success = false;
                $errors[] = $result['error'];
            }
            
            // Update webhook last triggered time
            Capsule::table('mod_{module}_webhooks')
                ->where('id', $handler['id'])
                ->update(['last_triggered' => date('Y-m-d H:i:s')]);
        }
        
        // Update event status
        Capsule::table('mod_{module}_webhook_events')
            ->where('id', $eventId)
            ->update([
                'status' => $success ? 'completed' : 'failed',
                'processed_at' => date('Y-m-d H:i:s'),
            ]);
        
        return [
            'success' => $success,
            'event_id' => $eventId,
            'errors' => $errors,
        ];
    }
    
    /**
     * Trigger webhook endpoint
     */
    private function triggerWebhook(array $handler, array $eventData): array {
        $url = $handler['url'];
        $headers = $handler['headers'];
        
        $payload = json_encode([
            'event' => $eventData['event'] ?? 'unknown',
            'timestamp' => date('c'),
            'data' => $eventData,
        ]);
        
        $ch = curl_init();
        
        $httpHeaders = [
            'Content-Type: application/json',
            'Accept: application/json',
            'X-Webhook-Event: ' . ($eventData['event'] ?? 'unknown'),
        ];
        
        // Add custom headers
        foreach ($headers as $key => $value) {
            $httpHeaders[] = "{$key}: {$value}";
        }
        
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_HTTPHEADER => $httpHeaders,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_CONNECTTIMEOUT => 10,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);
        
        // Log the request
        Capsule::table('mod_{module}_webhook_logs')->insert([
            'event_type' => $eventData['event'] ?? 'unknown',
            'payload' => $payload,
            'response' => is_string($response) ? substr($response, 0, 1000) : '',
            'status_code' => $httpCode,
            'success' => $httpCode >= 200 && $httpCode < 300,
            'error_message' => $error ?: null,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        if ($httpCode >= 200 && $httpCode < 300) {
            return ['success' => true];
        }
        
        return [
            'success' => false,
            'error' => "HTTP {$httpCode}: {$error}",
            'http_code' => $httpCode,
        ];
    }
    
    /**
     * Schedule async processing via cron
     */
    private function scheduleAsyncProcessing(int $eventId): void {
        Capsule::table('mod_{module}_webhook_events')
            ->where('id', $eventId)
            ->update([
                'next_attempt' => date('Y-m-d H:i:s'),
            ]);
    }
    
    /**
     * Process pending events (called by cron)
     */
    public function processPendingEvents(): int {
        $pending = Capsule::table('mod_{module}_webhook_events')
            ->where('status', 'pending')
            ->where('next_attempt', '<=', date('Y-m-d H:i:s'))
            ->limit(100)
            ->get();
        
        $processed = 0;
        
        foreach ($pending as $event) {
            $eventData = json_decode($event->payload, true);
            $result = $this->processEvent($event->id, $eventData);
            
            if ($result['success']) {
                $processed++;
            }
            
            // Update attempt count
            Capsule::table('mod_{module}_webhook_events')
                ->where('id', $event->id)
                ->update([
                    'attempts' => $event->attempts + 1,
                ]);
        }
        
        return $processed;
    }
}
```

## Event Handlers

```php
<?php
/**
 * Event Handler Implementations
 */

namespace WebhookListener;

class EventHandlers {
    
    /**
     * Handle ClientAdd event
     */
    public static function handleClientAdd(array $data): void {
        $clientId = $data['userid'] ?? 0;
        $email = $data['email'] ?? '';
        
        logActivity("Webhook: New client added - ID: {$clientId}, Email: {$email}");
        
        // Add custom logic here:
        // - Sync to external CRM
        // - Send welcome email
        // - Create account in external system
        // - Trigger automation workflows
    }
    
    /**
     * Handle InvoicePaid event
     */
    public static function handleInvoicePaid(array $data): void {
        $invoiceId = $data['invoiceid'] ?? 0;
        $amount = $data['total'] ?? 0;
        
        logActivity("Webhook: Invoice paid - ID: {$invoiceId}, Amount: {$amount}");
        
        // Add custom logic here:
        // - Update external accounting system
        // - Trigger fulfillment workflows
        // - Send notification to sales team
        // - Activate related service
    }
    
    /**
     * Handle AfterModuleCreate event
     */
    public static function handleAfterModuleCreate(array $data): void {
        $serviceId = $data['serviceid'] ?? 0;
        $params = $data['params'] ?? [];
        
        logActivity("Webhook: Service created - ID: {$serviceId}");
        
        // Add custom logic here:
        // - Configure DNS
        // - Set up monitoring
        // - Create backup job
        // - Initialize monitoring
    }
    
    /**
     * Handle TicketOpen event
     */
    public static function handleTicketOpen(array $data): void {
        $ticketId = $data['ticketid'] ?? 0;
        $subject = $data['subject'] ?? '';
        
        logActivity("Webhook: Ticket opened - ID: {$ticketId}, Subject: {$subject}");
        
        // Add custom logic here:
        // - Post to Slack/Teams
        // - Create Jira ticket
        // - Send SMS to support team
        // - Update external helpdesk
    }
    
    /**
     * Handle AcceptOrder event
     */
    public static function handleAcceptOrder(array $data): void {
        $orderId = $data['orderid'] ?? 0;
        
        logActivity("Webhook: Order accepted - ID: {$orderId}");
        
        // Add custom logic here:
        // - Send order confirmation
        // - Start provisioning
        // - Update inventory
        // - Trigger shipping notification
    }
    
    /**
     * Handle AfterModuleSuspend event
     */
    public static function handleAfterModuleSuspend(array $data): void {
        $serviceId = $data['serviceid'] ?? 0;
        
        logActivity("Webhook: Service suspended - ID: {$serviceId}");
        
        // Add custom logic here:
        // - Update monitoring
        // - Send notification
        // - Log suspension reason
    }
    
    /**
     * Handle AfterModuleTerminate event
     */
    public static function handleAfterModuleTerminate(array $data): void {
        $serviceId = $data['serviceid'] ?? 0;
        
        logActivity("Webhook: Service terminated - ID: {$serviceId}");
        
        // Add custom logic here:
        // - Clean up resources
        // - Archive data
        // - Send cancellation confirmation
        // - Update external systems
    }
}
```

## Admin Configuration Template

```smarty
<div class="webhook-listener">
    <div class="row">
        <div class="col-md-12">
            <div class="alert alert-info">
                <i class="fa fa-info-circle"></i>
                Configure webhook listener settings and manage webhook endpoints.
            </div>
        </div>
    </div>

    <form method="post" action="{$smarty.server.PHP_SELF}?action=module_settings&module={module}">
        <input type="hidden" name="csrf_token" value="{$csrf_token}">
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Security Settings</h3>
            </div>
            <div class="panel-body">
                <div class="form-group">
                    <label for="webhook_secret">Webhook Secret</label>
                    <input type="text" name="webhook_secret" id="webhook_secret" 
                           class="form-control" value="{$webhook_secret}"
                           placeholder="Enter secret for signature validation">
                    <span class="help-block">
                        Used to validate incoming webhook signatures via HMAC-SHA256
                    </span>
                </div>
                
                <div class="form-group">
                    <label for="allowed_ips">Allowed IP Addresses</label>
                    <textarea name="allowed_ips" id="allowed_ips" class="form-control" rows="3"
                              placeholder="1.2.3.4, 5.6.7.8">{$allowed_ips}</textarea>
                    <span class="help-block">
                        Comma-separated list of IP addresses allowed to send webhooks (leave empty to allow all)
                    </span>
                </div>
            </div>
        </div>
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Processing Settings</h3>
            </div>
            <div class="panel-body">
                <div class="form-group">
                    <label>
                        <input type="checkbox" name="async_processing" value="1" 
                               {$async_processing.checked}>
                        Enable Async Processing
                    </label>
                    <span class="help-block">
                        Process webhooks asynchronously via cron job for better performance
                    </span>
                </div>
                
                <div class="form-group">
                    <label for="max_retries">Max Retry Attempts</label>
                    <input type="number" name="max_retries" id="max_retries" 
                           class="form-control" value="{$max_retries|default:3}" min="0" max="10">
                </div>
            </div>
        </div>
        
        <button type="submit" class="btn btn-primary">
            <i class="fa fa-save"></i> Save Settings
        </button>
    </form>

    <hr>

    <div class="panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title">Webhook Endpoints</h3>
        </div>
        <div class="panel-body">
            <div class="mb-3">
                <a href="?module={module}&action=webhooks&sub=add" class="btn btn-success">
                    <i class="fa fa-plus"></i> Add Webhook Endpoint
                </a>
                <a href="?module={module}&action=test" class="btn btn-info">
                    <i class="fa fa-flask"></i> Test Webhook
                </a>
            </div>
            
            <table class="table table-striped">
                <thead>
                    <tr>
                        <th>Event Type</th>
                        <th>Webhook URL</th>
                        <th>Status</th>
                        <th>Last Triggered</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {foreach $webhooks as $webhook}
                    <tr>
                        <td><span class="label label-primary">{$webhook.event_type}</span></td>
                        <td><code>{$webhook.webhook_url}</code></td>
                        <td>
                            {if $webhook.is_active}
                                <span class="label label-success">Active</span>
                            {else}
                                <span class="label label-default">Inactive</span>
                            {/if}
                        </td>
                        <td>{$webhook.last_triggered}</td>
                        <td>
                            <a href="?module={module}&action=webhooks&sub=edit&id={$webhook.id}" 
                               class="btn btn-xs btn-default">
                                <i class="fa fa-edit"></i>
                            </a>
                            <a href="?module={module}&action=webhooks&sub=delete&id={$webhook.id}" 
                               class="btn btn-xs btn-danger" onclick="return confirm('Delete this webhook?')">
                                <i class="fa fa-trash"></i>
                            </a>
                        </td>
                    </tr>
                    {/foreach}
                </tbody>
            </table>
        </div>
    </div>

    <div class="panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title">Webhook URL</h3>
        </div>
        <div class="panel-body">
            <div class="alert alert-success">
                <strong>Your Webhook Endpoint:</strong><br>
                <code>{$base_url}modules/addons/{module}/webhook.php</code>
            </div>
            <p>Use this URL in WHMCS webhook configuration to receive events.</p>
        </div>
    </div>
</div>
```

## Cron Processing Hook

```php
<?php
/**
 * Webhook Cron Processing Hook
 * 
 * Add this to your hooks.php to process pending webhooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Process pending webhook events
add_hook('DailyCronJob', 1, function($vars) {
    $processor = new \WebhookListener\EventProcessor();
    $processed = $processor->processPendingEvents();
    
    logActivity("Webhook processor: Processed {$processed} pending events");
});
```

## Checklist

```
Pre-Dev:
□ Define webhook events to handle
□ Plan validation requirements
□ Design retry mechanism
□ Identify external endpoints
□ Plan async vs sync processing

Development:
□ Create webhook endpoint handler
□ Implement WebhookValidator class
□ Implement EventProcessor class
□ Create event handler implementations
□ Add webhook registration system
□ Implement retry mechanism
□ Add logging system
□ Build admin configuration UI
□ Create async processing hook
□ Add cron job for pending events

Testing:
□ Test webhook endpoint
□ Verify signature validation
□ Test IP whitelist
□ Test all event handlers
□ Verify retry mechanism
□ Test async processing
□ Test error handling
□ Verify logging
□ Test with WHMCS webhook
```