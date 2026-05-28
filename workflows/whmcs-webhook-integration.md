# WHMCS Webhook Integration Workflow

## Purpose

Comprehensive guide to implementing and managing webhook integrations in WHMCS for real-time event notifications, external system synchronization, and automation workflows.

## Prerequisites

- WHMCS installation with API access
- Understanding of HTTP webhooks and REST APIs
- SSL certificate for secure webhook endpoints
- Access to external service webhook settings

## Workflow Steps

### Step 1: Webhook Endpoint Implementation

Create secure webhook endpoints for receiving events:

```php
// modules/addons/webhook_receiver/webhook_receiver.php

add_hook('AdminConfigSidebar', 1, function($vars) {
    return [
        'WebhookLogs' => [
            'uri' => 'webhook_receiver/logs',
            'label' => 'Webhook Logs',
            'icon' => 'fa-plug',
        ],
    ];
});

/**
 * Main webhook receiver class
 */
class WebhookReceiver
{
    private $db;
    private $secretKey;
    private $maxRetries = 3;
    private $retryDelay = 60; // seconds
    
    public function __construct($secretKey = null)
    {
        $this->secretKey = $secretKey ?? getWHMCSConfig('API Secret');
        $this->db = Capsule::connection();
    }
    
    /**
     * Process incoming webhook request
     */
    public function handleRequest(): array
    {
        // Get raw input for signature verification
        $rawInput = file_get_contents('php://input');
        
        // Get request headers
        $headers = $this->getRequestHeaders();
        
        // Verify webhook signature
        if (!$this->verifySignature($rawInput, $headers)) {
            $this->logWebhook('invalid_signature', $rawInput, $headers, 401);
            http_response_code(401);
            return ['error' => 'Invalid signature'];
        }
        
        // Parse webhook payload
        $payload = json_decode($rawInput, true);
        
        if (json_last_error() !== JSON_ERROR_NONE) {
            $this->logWebhook('invalid_json', $rawInput, $headers, 400);
            http_response_code(400);
            return ['error' => 'Invalid JSON payload'];
        }
        
        // Extract event type
        $eventType = $headers['X-Webhook-Event'] 
            ?? $payload['event'] ?? $payload['type'] ?? 'unknown';
        
        // Log the webhook
        $webhookId = $this->logWebhook($eventType, $rawInput, $headers, 200);
        
        // Process asynchronously
        $this->queueForProcessing($webhookId, $eventType, $payload);
        
        // Return success immediately
        return ['status' => 'received', 'id' => $webhookId];
    }
    
    /**
     * Verify webhook signature using HMAC
     */
    private function verifySignature(string $payload, array $headers): bool
    {
        $signature = $headers['X-Webhook-Signature'] ?? '';
        
        if (empty($signature)) {
            return false;
        }
        
        // Support both SHA256 and SHA512
        if (strpos($signature, 'sha256=') === 0) {
            $algorithm = 'sha256';
            $hash = substr($signature, 7);
        } elseif (strpos($signature, 'sha512=') === 0) {
            $algorithm = 'sha512';
            $hash = substr($signature, 7);
        } else {
            return false;
        }
        
        $expected = hash_hmac($algorithm, $payload, $this->secretKey);
        
        return hash_equals($expected, $hash);
    }
    
    /**
     * Get all request headers
     */
    private function getRequestHeaders(): array
    {
        $headers = [];
        
        foreach ($_SERVER as $key => $value) {
            if (strpos($key, 'HTTP_') === 0) {
                $header = str_replace('_', '-', substr($key, 5));
                $header = ucwords(strtolower($header), '-');
                $headers[$header] = $value;
            }
        }
        
        return $headers;
    }
    
    /**
     * Log webhook for debugging
     */
    private function logWebhook(string $event, string $payload, array $headers, int $status): int
    {
        return Capsule::table('mod_webhook_logs')->insertGetId([
            'event_type' => $event,
            'payload' => $payload,
            'headers' => json_encode($headers),
            'status_code' => $status,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    /**
     * Queue webhook for async processing
     */
    private function queueForProcessing(int $webhookId, string $eventType, array $payload): void
    {
        Capsule::table('mod_webhook_queue')->insert([
            'webhook_id' => $webhookId,
            'event_type' => $eventType,
            'payload' => json_encode($payload),
            'status' => 'pending',
            'attempts' => 0,
            'next_retry' => date('Y-m-d H:i:s'),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * Webhook processing cron job
 */
add_hook('DailyCronJob', 1, function($vars) {
    $webhooks = Capsule::table('mod_webhook_queue')
        ->where('status', 'pending')
        ->where('next_retry', '<=', date('Y-m-d H:i:s'))
        ->where('attempts', '<', 3)
        ->limit(100)
        ->get();
    
    foreach ($webhooks as $job) {
        processWebhookJob($job);
    }
});

function processWebhookJob($job): void
{
    $processor = new WebhookProcessor();
    $result = $processor->process($job->event_type, json_decode($job->payload, true));
    
    if ($result['success']) {
        Capsule::table('mod_webhook_queue')
            ->where('id', $job->id)
            ->update(['status' => 'completed', 'processed_at' => date('Y-m-d H:i:s')]);
    } else {
        $attempts = $job->attempts + 1;
        Capsule::table('mod_webhook_queue')
            ->where('id', $job->id)
            ->update([
                'attempts' => $attempts,
                'last_error' => $result['error'],
                'next_retry' => date('Y-m-d H:i:s', strtotime('+1 hour')),
                'status' => $attempts >= 3 ? 'failed' : 'pending',
            ]);
    }
}

/**
 * Event type handlers
 */
class WebhookProcessor
{
    private $handlers = [];
    
    public function __construct()
    {
        $this->registerHandlers();
    }
    
    private function registerHandlers(): void
    {
        $this->handlers = [
            'invoice.paid' => [$this, 'handleInvoicePaid'],
            'order.created' => [$this, 'handleOrderCreated'],
            'service.created' => [$this, 'handleServiceCreated'],
            'service.suspended' => [$this, 'handleServiceSuspended'],
            'service.terminated' => [$this, 'handleServiceTerminated'],
            'domain.transfer_completed' => [$this, 'handleDomainTransfer'],
            'ticket.created' => [$this, 'handleTicketCreated'],
            'client.created' => [$this, 'handleClientCreated'],
        ];
    }
    
    public function process(string $eventType, array $payload): array
    {
        if (!isset($this->handlers[$eventType])) {
            return ['success' => true, 'message' => 'No handler for event type'];
        }
        
        try {
            $handler = $this->handlers[$eventType];
            $handler($payload);
            return ['success' => true];
        } catch (Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }
    
    private function handleInvoicePaid(array $payload): void
    {
        $invoiceId = $payload['invoice_id'] ?? $payload['id'];
        
        // LogActivity("Webhook: Invoice {$invoiceId} marked as paid");
        
        // Trigger custom actions
        doHook('WebhookInvoicePaid', $payload);
    }
    
    private function handleOrderCreated(array $payload): void
    {
        $orderId = $payload['order_id'] ?? $payload['id'];
        
        doHook('WebhookOrderCreated', $payload);
    }
    
    private function handleServiceCreated(array $payload): void
    {
        $serviceId = $payload['service_id'] ?? $payload['id'];
        
        doHook('WebhookServiceCreated', $payload);
    }
}
```

### Step 2: Outbound Webhook Configuration

Configure WHMCS to send webhooks to external services:

```php
// modules/addons/webhook_sender/webhook_sender.php

/**
 * Outbound webhook sender class
 */
class WebhookSender
{
    private $webhookUrl;
    private $secretKey;
    private $timeout = 30;
    
    public function __construct($webhookUrl, $secretKey = null)
    {
        $this->webhookUrl = $webhookUrl;
        $this->secretKey = $secretKey ?? getWHMCSConfig('API Secret');
    }
    
    /**
     * Send webhook with signature
     */
    public function send(string $eventType, array $payload): array
    {
        $jsonPayload = json_encode($payload);
        $signature = hash_hmac('sha256', $jsonPayload, $this->secretKey);
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->webhookUrl,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $jsonPayload,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-Webhook-Event: ' . $eventType,
                'X-Webhook-Signature: sha256=' . $signature,
                'X-Webhook-Timestamp: ' . time(),
            ],
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);
        
        // Log the webhook
        $this->logOutboundWebhook($eventType, $payload, $httpCode, $response);
        
        if ($error) {
            throw new Exception("Webhook failed: {$error}");
        }
        
        if ($httpCode < 200 || $httpCode >= 300) {
            throw new Exception("Webhook returned HTTP {$httpCode}");
        }
        
        return json_decode($response, true) ?? ['success' => true];
    }
    
    private function logOutboundWebhook(string $event, array $payload, int $code, string $response): void
    {
        Capsule::table('mod_webhook_outbound')->insert([
            'event_type' => $event,
            'payload' => json_encode($payload),
            'response_code' => $code,
            'response' => substr($response, 0, 1000),
            'sent_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * Register webhook event hooks
 */
add_hook('InvoicePaid', 1, function($vars) {
    $webhook = new WebhookSender(
        Capsule::table('mod_webhook_endpoints')
            ->where('event', 'invoice.paid')
            ->where('enabled', 1)
            ->value('url')
    );
    
    $webhook->send('invoice.paid', [
        'event' => 'invoice.paid',
        'timestamp' => time(),
        'invoice_id' => $vars['invoice_id'],
        'amount' => $vars['amount'],
        'client_id' => $vars['user_id'],
    ]);
});

add_hook('AfterModuleCreate', 1, function($vars) {
    $webhook = new WebhookSender(
        Capsule::table('mod_webhook_endpoints')
            ->where('event', 'service.created')
            ->where('enabled', 1)
            ->value('url')
    );
    
    $webhook->send('service.created', [
        'event' => 'service.created',
        'timestamp' => time(),
        'service_id' => $vars['service_id'],
        'domain' => $vars['domain'],
        'module' => $vars['module'],
    ]);
});
```

### Step 3: Webhook Security Configuration

Implement security measures for webhook endpoints:

```php
// modules/addons/webhook_receiver/security.php

/**
 * Webhook IP whitelist and blacklist
 */
class WebhookSecurity
{
    private static $whitelist = [];
    private static $blacklist = [];
    
    public static function init(): void
    {
        // Load from configuration
        self::$whitelist = explode(',', Capsule::table('tblconfiguration')
            ->where('setting', 'webhook_ip_whitelist')
            ->value('value') ?: '');
        
        self::$blacklist = explode(',', Capsule::table('tblconfiguration')
            ->where('setting', 'webhook_ip_blacklist')
            ->value('value') ?: '');
    }
    
    /**
     * Validate incoming IP address
     */
    public static function validateIP(string $ip): bool
    {
        self::init();
        
        // Check blacklist first
        if (in_array($ip, self::$blacklist)) {
            return false;
        }
        
        // If whitelist is empty, allow all
        if (empty(self::$whitelist)) {
            return true;
        }
        
        // Check whitelist
        return in_array($ip, self::$whitelist);
    }
    
    /**
     * Validate webhook timestamp (prevent replay attacks)
     */
    public static function validateTimestamp(string $timestamp, int $tolerance = 300): bool
    {
        $diff = abs(time() - (int)$timestamp);
        return $diff <= $tolerance;
    }
    
    /**
     * Rate limiting for webhook endpoints
     */
    public static function checkRateLimit(string $ip, int $maxRequests = 100, int $window = 60): bool
    {
        $cacheKey = "webhook_rate_{$ip}";
        $cache = \WHMCS\TransientData::getInstance();
        
        $requests = (int) $cache->retrieve($cacheKey) ?: 0;
        
        if ($requests >= $maxRequests) {
            return false;
        }
        
        $cache->store($cacheKey, $requests + 1, $window);
        
        return true;
    }
}

/**
 * Security validation in webhook handler
 */
add_hook('WebhookRequest', 1, function($vars) {
    $ip = $_SERVER['REMOTE_ADDR'] ?? '';
    
    if (!WebhookSecurity::validateIP($ip)) {
        return ['allow' => false, 'reason' => 'IP not allowed'];
    }
    
    if (!WebhookSecurity::checkRateLimit($ip)) {
        return ['allow' => false, 'reason' => 'Rate limit exceeded'];
    }
    
    $timestamp = $_SERVER['HTTP_X_WEBHOOK_TIMESTAMP'] ?? '';
    if ($timestamp && !WebhookSecurity::validateTimestamp($timestamp)) {
        return ['allow' => false, 'reason' => 'Request timestamp too old'];
    }
    
    return ['allow' => true];
});
```

### Step 4: Webhook Admin Interface

Create admin interface for webhook management:

```php
// modules/addons/webhook_receiver/admin.php

function webhook_receiver_config(): array
{
    return [
        'name' => 'Webhook Receiver',
        'description' => 'Receive and process webhooks from external services',
        'version' => '1.0',
        'author' => 'Your Name',
    ];
}

function webhook_receiver_activate(): array
{
    Capsule::schema()->create('mod_webhook_logs', function($table) {
        $table->increments('id');
        $table->string('event_type', 100);
        $table->longText('payload');
        $table->text('headers');
        $table->integer('status_code');
        $table->string('ip_address', 45);
        $table->timestamp('created_at')->useCurrent();
    });
    
    Capsule::schema()->create('mod_webhook_queue', function($table) {
        $table->increments('id');
        $table->integer('webhook_id');
        $table->string('event_type', 100);
        $table->longText('payload');
        $table->enum('status', ['pending', 'processing', 'completed', 'failed']);
        $table->integer('attempts')->default(0);
        $table->text('last_error');
        $table->timestamp('next_retry');
        $table->timestamp('processed_at')->nullable();
        $table->timestamp('created_at')->useCurrent();
    });
    
    Capsule::schema()->create('mod_webhook_outbound', function($table) {
        $table->increments('id');
        $table->string('event_type', 100);
        $table->longText('payload');
        $table->integer('response_code');
        $table->text('response');
        $table->timestamp('sent_at')->useCurrent();
    });
    
    return ['status' => 'success'];
}

function webhook_receiver_deactivate(): array
{
    Capsule::schema()->dropIfExists('mod_webhook_logs');
    Capsule::schema()->dropIfExists('mod_webhook_queue');
    Capsule::schema()->dropIfExists('mod_webhook_outbound');
    
    return ['status' => 'success'];
}

function webhook_reducer_output(array $vars): void
{
    $action = $_REQUEST['action'] ?? 'logs';
    
    switch ($action) {
        case 'logs':
            echo webhook_receiver_render_logs();
            break;
        case 'view':
            echo webhook_receiver_render_detail($_GET['id']);
            break;
        case 'retry':
            webhook_receiver_retry($_GET['id']);
            redir('action=logs');
            break;
        case 'settings':
            echo webhook_receiver_render_settings();
            break;
    }
}

function webhook_receiver_render_logs(): string
{
    $logs = Capsule::table('mod_webhook_logs')
        ->orderBy('created_at', 'desc')
        ->limit(100)
        ->get();
    
    $html = '<div class="webhook-logs">';
    $html .= '<h2>Webhook Logs</h2>';
    $html .= '<table class="datatable"><thead><tr>';
    $html .= '<th>ID</th><th>Event</th><th>Status</th><th>IP</th><th>Date</th><th>Actions</th>';
    $html .= '</tr></thead><tbody>';
    
    foreach ($logs as $log) {
        $html .= '<tr>';
        $html .= '<td>' . $log->id . '</td>';
        $html .= '<td>' . htmlspecialchars($log->event_type) . '</td>';
        $html .= '<td><span class="label label-' . ($log->status_code < 400 ? 'success' : 'danger') . '">' . $log->status_code . '</span></td>';
        $html .= '<td>' . htmlspecialchars($log->ip_address) . '</td>';
        $html .= '<td>' . $log->created_at . '</td>';
        $html .= '<td><a href="?action=view&id=' . $log->id . '">View</a></td>';
        $html .= '</tr>';
    }
    
    $html .= '</tbody></table></div>';
    
    return $html;
}
```

## Best Practices

1. **Always verify signatures** - Prevent spoofed webhook requests
2. **Process asynchronously** - Return 200 quickly, process in background
3. **Implement idempotency** - Handle duplicate webhook deliveries gracefully
4. **Use HTTPS endpoints** - Encrypt all webhook communication
5. **Implement retry logic** - Queue failed webhooks for retry
6. **Log all webhooks** - Maintain audit trail for debugging
7. **Rate limit** - Protect against abuse and DDoS
8. **Validate timestamps** - Prevent replay attacks

## Common Pitfalls to Avoid

1. **Not verifying signatures** - Allows spoofed requests
2. **Synchronous processing** - Causes timeouts on slow handlers
3. **No retry mechanism** - Lost webhooks cause data inconsistency
4. **Logging sensitive data** - Never log passwords or payment details
5. **Ignoring rate limits** - Gets IP blocked by providers
6. **Missing error handling** - Fails silently on exceptions
7. **Processing in admin context** - Can cause session issues
8. **Not implementing idempotency** - Duplicate processing causes bugs
