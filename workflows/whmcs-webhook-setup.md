# WHMCS Webhook Setup and Handling Workflow

## Purpose

Comprehensive guide to setting up and handling webhooks in WHMCS. Covers webhook registration, signature verification, event processing, security best practices, and error handling.

## Prerequisites

- WHMCS 7.0 or higher
- SSL certificate
- Understanding of HTTP protocols
- Basic PHP knowledge
- External service with webhook support (optional)

## Workflow Steps

### Step 1: Understanding WHMCS Webhook System

WHMCS includes a native webhook system for outgoing webhooks:

```php
// WHMCS Native Webhook Registration
// Configuration > System Settings > Webhooks

/**
 * WHMCS Webhook Events Reference:
 *
 * Account Events:
 * - AccountCreate - After client account creation
 * - AccountUpdate - After client account update
 * - AccountDelete - Before client account deletion
 *
 * Service Events:
 * - AfterModuleCreate - After service provisioning
 * - AfterModuleSuspend - After service suspension
 * - AfterModuleUnsuspend - After service unsuspension
 * - AfterModuleTerminate - After service termination
 * - AfterModuleChangePassword - After password change
 *
 * Invoice Events:
 * - InvoicePaid - After invoice payment
 * - InvoiceCancelled - After invoice cancellation
 * - InvoiceDeleted - After invoice deletion
 * - InvoicePaymentRejected - Payment rejection
 *
 * Domain Events:
 * - DomainRegister - After domain registration
 * - DomainTransferCompleted - After transfer completion
 * - DomainRenewed - After domain renewal
 *
 * Support Events:
 * - TicketOpen - When support ticket opened
 * - TicketReply - When ticket replied
 * - TicketClose - When ticket closed
 *
 * Other Events:
 * - DailyCronJob - Daily cron execution
 * - AfterCalculatingCartTotals - After cart calculation
 */
```

### Step 2: Creating Inbound Webhook Handler

Create a secure webhook endpoint for receiving external events:

```php
// modules/addons/youraddon/webhooks.php

/**
 * WHMCS Inbound Webhook Handler
 *
 * Secure endpoint for receiving external webhooks.
 * Includes signature verification, retry handling, and event logging.
 */
add_hook('RouteAccept', 1, function($vars) {
    $routes = $vars['routes'];

    // Register webhook routes
    $routes['api/webhook/{provider}'] = [
        'path' => '/api/webhook/{provider}',
        'controller' => 'WebhookController',
        'action' => 'handle',
        'method' => ['POST', 'OPTIONS'],
        'authentication' => 'none',
    ];

    return ['routes' => $routes];
});

/**
 * Webhook Handler Class
 */
class WebhookController
{
    private $webhookSecret;
    private $logTable = 'mod_webhook_events';

    public function __construct()
    {
        // Load webhook secret from configuration
        $this->webhookSecret = \Di::make('config')
            ->get('webhook_secret', '');
    }

    /**
     * Handle incoming webhook request
     */
    public function handle(): array
    {
        $requestBody = file_get_contents('php://input');
        $headers = $this->getRequestHeaders();
        $provider = $vars['params']['provider'] ?? 'unknown';

        // CORS preflight
        if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
            return [
                'status' => 204,
                'headers' => [
                    'Access-Control-Allow-Origin' => '*',
                    'Access-Control-Allow-Methods' => 'POST, OPTIONS',
                    'Access-Control-Allow-Headers' => 'Content-Type, X-Signature, X-Timestamp',
                ],
            ];
        }

        // Verify signature
        if (!$this->verifySignature($requestBody, $headers)) {
            $this->logEvent($provider, 'verification_failed', [], $headers);
            return [
                'status' => 401,
                'body' => ['error' => 'Invalid signature'],
            ];
        }

        // Parse payload
        $payload = json_decode($requestBody, true);
        if (json_last_error() !== JSON_ERROR_NONE) {
            return [
                'status' => 400,
                'body' => ['error' => 'Invalid JSON payload'],
            ];
        }

        // Log event
        $eventId = $this->logEvent($provider, 'received', $payload, $headers);

        // Process event asynchronously if needed
        $result = $this->processEvent($provider, $payload, $headers);

        if ($result['success']) {
            $this->markEventProcessed($eventId);
        }

        return [
            'status' => 200,
            'body' => [
                'success' => true,
                'event_id' => $eventId,
            ],
        ];
    }

    /**
     * Verify webhook signature using HMAC
     */
    private function verifySignature(string $payload, array $headers): bool
    {
        $signatureHeader = $headers['x-signature'] ?? $headers['x-hub-signature-256'] ?? '';

        // Handle GitHub-style sha256= prefix
        if (strpos($signatureHeader, 'sha256=') === 0) {
            $signatureHeader = substr($signatureHeader, 7);
        }

        if (empty($signatureHeader)) {
            return false;
        }

        // Compute expected signature
        $expectedSignature = hash_hmac('sha256', $payload, $this->webhookSecret);

        // Timing-safe comparison to prevent timing attacks
        return hash_equals($expectedSignature, $signatureHeader);
    }

    /**
     * Process webhook event based on provider and type
     */
    private function processEvent(string $provider, array $payload, array $headers): array
    {
        $eventType = $this->determineEventType($provider, $payload, $headers);

        try {
            switch ($provider) {
                case 'stripe':
                    return $this->handleStripeWebhook($eventType, $payload);
                case 'paypal':
                    return $this->handlePayPalWebhook($eventType, $payload);
                case 'inventory':
                    return $this->handleInventoryWebhook($eventType, $payload);
                case 'crm':
                    return $this->handleCRMWebhook($eventType, $payload);
                default:
                    return $this->handleGenericWebhook($provider, $eventType, $payload);
            }
        } catch (\Exception $e) {
            logActivity("Webhook processing error [{$provider}/{$eventType}]: " . $e->getMessage());
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    /**
     * Determine event type from headers or payload
     */
    private function determineEventType(string $provider, array $payload, array $headers): string
    {
        // Check various header formats
        $eventType = $headers['x-event-type'] ??
                     $headers['x-github-event'] ??
                     $headers['x-stripe-event'] ??
                     $payload['type'] ??
                     $payload['event'] ??
                     'unknown';

        return strtolower($eventType);
    }

    /**
     * Handle Stripe webhooks
     */
    private function handleStripeWebhook(string $eventType, array $payload): array
    {
        switch ($eventType) {
            case 'invoice.paid':
                return $this->handleStripeInvoicePaid($payload);
            case 'payment_intent.succeeded':
                return $this->handleStripePaymentSucceeded($payload);
            case 'charge.refunded':
                return $this->handleStripeRefund($payload);
            case 'customer.subscription.updated':
                return $this->handleStripeSubscriptionUpdated($payload);
            case 'customer.subscription.deleted':
                return $this->handleStripeSubscriptionDeleted($payload);
            default:
                logActivity("Unhandled Stripe event: {$eventType}");
                return ['success' => true]; // Acknowledge but don't process
        }
    }

    /**
     * Process Stripe invoice paid event
     */
    private function handleStripeInvoicePaid(array $payload): array
    {
        $stripeInvoiceId = $payload['data']['object']['id'] ?? '';
        $stripeCustomerId = $payload['data']['object']['customer'] ?? '';
        $amountPaid = ($payload['data']['object']['amount_paid'] ?? 0) / 100;

        if (empty($stripeInvoiceId)) {
            return ['success' => false, 'error' => 'Missing Stripe invoice ID'];
        }

        // Find matching WHMCS invoice
        $invoice = Capsule::table('tblinvoices')
            ->where('status', 'Unpaid')
            ->where(function ($query) use ($stripeInvoiceId) {
                $query->where('notes', 'like', '%' . $stripeInvoiceId . '%')
                      ->orWhere('notes', 'like', '%stripe_%');
            })
            ->first();

        if (!$invoice) {
            // Try to match by amount and date
            $invoice = Capsule::table('tblinvoices')
                ->where('status', 'Unpaid')
                ->where('total', $amountPaid)
                ->orderBy('date', 'desc')
                ->first();
        }

        if ($invoice) {
            $transactionId = 'stripe_' . $stripeInvoiceId;

            addInvoicePayment(
                $invoice->id,
                $transactionId,
                $amountPaid,
                0,
                'stripe'
            );

            return ['success' => true, 'invoice_id' => $invoice->id];
        }

        return ['success' => false, 'error' => 'Invoice not found'];
    }

    /**
     * Handle PayPal webhooks
     */
    private function handlePayPalWebhook(string $eventType, array $payload): array
    {
        switch ($eventType) {
            case 'PAYMENT.CAPTURE.COMPLETED':
                return $this->handlePayPalPaymentCompleted($payload);
            case 'PAYMENT.CAPTURE.REFUNDED':
                return $this->handlePayPalRefund($payload);
            default:
                return ['success' => true];
        }
    }

    /**
     * Handle inventory webhooks
     */
    private function handleInventoryWebhook(string $eventType, array $payload): array
    {
        $productId = $payload['product_id'] ?? null;
        $stockLevel = $payload['stock'] ?? null;

        if ($productId && $stockLevel !== null) {
            Capsule::table('mod_product_inventory')
                ->updateOrInsert(
                    ['product_id' => $productId],
                    [
                        'stock_level' => $stockLevel,
                        'updated_at' => date('Y-m-d H:i:s'),
                        'source' => 'webhook',
                    ]
                );
        }

        return ['success' => true];
    }

    /**
     * Handle generic webhooks
     */
    private function handleGenericWebhook(string $provider, string $eventType, array $payload): array
    {
        // Log for admin review
        logActivity("Webhook [{$provider}] event [{$eventType}]: " . json_encode($payload));

        return ['success' => true];
    }

    /**
     * Get all request headers
     */
    private function getRequestHeaders(): array
    {
        $headers = [];
        $ignoreHeaders = ['HOST', 'CONTENT_LENGTH', 'CONTENT_TYPE'];

        foreach ($_SERVER as $key => $value) {
            if (strpos($key, 'HTTP_') === 0) {
                $headerName = str_replace('_', '-', substr($key, 5));
                $headers[strtolower($headerName)] = $value;
            }
        }

        // Handle Content-Type header
        if (isset($_SERVER['CONTENT_TYPE'])) {
            $headers['content-type'] = $_SERVER['CONTENT_TYPE'];
        }

        return $headers;
    }

    /**
     * Log webhook event for auditing
     */
    private function logEvent(string $provider, string $status, array $payload, array $headers, string $error = ''): int
    {
        $this->ensureLogTableExists();

        return Capsule::table($this->logTable)->insertGetId([
            'provider' => $provider,
            'event_type' => $headers['x-event-type'] ?? 'unknown',
            'status' => $status,
            'payload' => json_encode($payload),
            'headers' => json_encode($headers),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'error_message' => $error,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Create log table if not exists
     */
    private function ensureLogTableExists(): void
    {
        if (!Capsule::schema()->hasTable($this->logTable)) {
            Capsule::schema()->create($this->logTable, function($table) {
                $table->increments('id');
                $table->string('provider', 100);
                $table->string('event_type', 100);
                $table->string('status', 50);
                $table->longText('payload');
                $table->text('headers');
                $table->string('ip_address', 50);
                $table->text('error_message')->nullable();
                $table->timestamp('created_at')->useCurrent();
                $table->timestamp('processed_at')->nullable();
                $table->index('created_at');
            });
        }
    }

    private function markEventProcessed(int $eventId): void
    {
        if ($eventId > 0) {
            Capsule::table($this->logTable)
                ->where('id', $eventId)
                ->update(['processed_at' => date('Y-m-d H:i:s')]);
        }
    }

    private function handleStripePaymentSucceeded(array $payload): array { /* ... */ return ['success' => true]; }
    private function handleStripeRefund(array $payload): array { /* ... */ return ['success' => true]; }
    private function handleStripeSubscriptionUpdated(array $payload): array { /* ... */ return ['success' => true]; }
    private function handleStripeSubscriptionDeleted(array $payload): array { /* ... */ return ['success' => true]; }
    private function handlePayPalPaymentCompleted(array $payload): array { /* ... */ return ['success' => true]; }
    private function handlePayPalRefund(array $payload): array { /* ... */ return ['success' => true]; }
    private function handleCRMWebhook(string $eventType, array $payload): array { /* ... */ return ['success' => true]; }
}
```

### Step 3: Setting Up WHMCS Outbound Webhooks

Configure outbound webhooks for WHMCS events:

```php
// modules/addons/youraddon/includes/webhook_config.php

/**
 * WHMCS Outbound Webhook Configuration
 *
 * Programmatically configure outbound webhooks for WHMCS events.
 */
class WHMCS_OutboundWebhook_Config
{
    private $webhookUrl;
    private $webhookSecret;

    public function __construct(string $webhookUrl, string $webhookSecret)
    {
        $this->webhookUrl = $webhookUrl;
        $this->webhookSecret = $webhookSecret;
    }

    /**
     * Register webhook for event
     */
    public function registerWebhook(string $eventName, array $settings = []): bool
    {
        $webhook = Capsule::table('tblwebhooks')
            ->where('url', $this->webhookUrl)
            ->where('event', $eventName)
            ->first();

        if ($webhook) {
            // Update existing
            Capsule::table('tblwebhooks')
                ->where('id', $webhook->id)
                ->update([
                    'settings' => json_encode($settings),
                    'updated_at' => date('Y-m-d H:i:s'),
                ]);
        } else {
            // Create new
            Capsule::table('tblwebhooks')->insert([
                'name' => $settings['name'] ?? $eventName,
                'url' => $this->webhookUrl,
                'event' => $eventName,
                'settings' => json_encode($settings),
                'created_at' => date('Y-m-d H:i:s'),
            ]);
        }

        return true;
    }

    /**
     * Register multiple webhooks for events
     */
    public function registerMultipleWebhooks(array $events): void
    {
        foreach ($events as $eventName => $settings) {
            $this->registerWebhook($eventName, $settings);
        }
    }

    /**
     * Send outbound webhook
     */
    public function sendWebhook(string $eventName, array $data): array
    {
        $payload = json_encode([
            'event' => $eventName,
            'timestamp' => time(),
            'data' => $data,
        ]);

        $signature = hash_hmac('sha256', $payload, $this->webhookSecret);

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->webhookUrl,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-Signature: ' . $signature,
                'X-Event: ' . $eventName,
                'X-Timestamp: ' . time(),
            ],
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception("Webhook failed: {$error}");
        }

        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'http_code' => $httpCode,
            'response' => $response,
        ];
    }
}

/**
 * Hook into WHMCS events and send webhooks
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $webhook = new WHMCS_OutboundWebhook_Config(
        'https://your-external-service.com/webhook',
        'your_webhook_secret'
    );

    try {
        $webhook->sendWebhook('module.created', [
            'service_id' => $vars['serviceid'],
            'server_id' => $vars['serverid'],
            'user_id' => $vars['userid'],
            'domain' => $vars['domain'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    } catch (\Exception $e) {
        logActivity("Webhook send failed: " . $e->getMessage());
    }
});

add_hook('InvoicePaid', 1, function($vars) {
    $webhook = new WHMCS_OutboundWebhook_Config(
        'https://your-external-service.com/webhook',
        'your_webhook_secret'
    );

    try {
        $webhook->sendWebhook('invoice.paid', [
            'invoice_id' => $vars['invoiceid'],
            'user_id' => $vars['userid'],
            'total' => $vars['total'],
            'paid_at' => date('Y-m-d H:i:s'),
        ]);
    } catch (\Exception $e) {
        logActivity("Webhook send failed: " . $e->getMessage());
    }
});

add_hook('TicketOpen', 1, function($vars) {
    $webhook = new WHMCS_OutboundWebhook_Config(
        'https://your-external-service.com/webhook',
        'your_webhook_secret'
    );

    try {
        $webhook->sendWebhook('ticket.created', [
            'ticket_id' => $vars[' ticketid'],
            'subject' => $vars['subject'],
            'priority' => $vars['priority'],
            'user_id' => $vars['userid'],
        ]);
    } catch (\Exception $e) {
        logActivity("Webhook send failed: " . $e->getMessage());
    }
});
```

### Step 4: Implementing Retry Logic

Handle webhook delivery failures with retry logic:

```php
// modules/addons/youraddon/includes/webhook_retry.php

/**
 * Webhook Retry Handler
 *
 * Manages retry attempts for failed webhook deliveries.
 */
class WHMCS_Webhook_Retry_Handler
{
    private $maxRetries = 5;
    private $baseDelay = 60; // 1 minute
    private $maxDelay = 3600; // 1 hour
    private $retryTable = 'mod_webhook_retries';

    public function __construct()
    {
        $this->ensureRetryTableExists();
    }

    /**
     * Schedule webhook for retry
     */
    public function scheduleRetry(array $webhookData, int $attempt = 1): void
    {
        if ($attempt > $this->maxRetries) {
            $this->markFailed($webhookData);
            return;
        }

        // Calculate delay with exponential backoff
        $delay = min($this->baseDelay * pow(2, $attempt - 1), $this->maxDelay);
        $scheduledAt = $this->randomizeDelay($delay);

        Capsule::table($this->retryTable)->insert([
            'webhook_url' => $webhookData['url'],
            'event_type' => $webhookData['event_type'],
            'payload' => $webhookData['payload'],
            'headers' => json_encode($webhookData['headers'] ?? []),
            'attempt' => $attempt,
            'scheduled_at' => $scheduledAt,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Process scheduled retries (called by cron)
     */
    public function processScheduledRetries(): int
    {
        $pendingRetries = Capsule::table($this->retryTable)
            ->where('scheduled_at', '<=', date('Y-m-d H:i:s'))
            ->where('status', 'pending')
            ->get();

        $processed = 0;

        foreach ($pendingRetries as $retry) {
            try {
                $this->sendWebhookWithRetry($retry);

                Capsule::table($this->retryTable)
                    ->where('id', $retry->id)
                    ->update([
                        'status' => 'completed',
                        'completed_at' => date('Y-m-d H:i:s'),
                    ]);

                $processed++;
            } catch (\Exception $e) {
                // Schedule next retry or mark as failed
                $nextAttempt = ($retry->attempt || 0) + 1;

                if ($nextAttempt > $this->maxRetries) {
                    Capsule::table($this->retryTable)
                        ->where('id', $retry->id)
                        ->update([
                            'status' => 'failed',
                            'error_message' => $e->getMessage(),
                            'completed_at' => date('Y-m-d H:i:s'),
                        ]);

                    logActivity("Webhook retry exhausted for event: {$retry->event_type}");
                } else {
                    $this->scheduleRetry([
                        'url' => $retry->webhook_url,
                        'event_type' => $retry->event_type,
                        'payload' => $retry->payload,
                        'headers' => json_decode($retry->headers, true),
                    ], $nextAttempt);

                    Capsule::table($this->retryTable)
                        ->where('id', $retry->id)
                        ->update(['status' => 'superseded']);
                }
            }
        }

        return $processed;
    }

    /**
     * Send webhook with retry tracking
     */
    private function sendWebhookWithRetry(stdClass $retry): void
    {
        $headers = json_decode($retry->headers, true);
        $headers['X-Retry-Attempt'] = $retry->attempt;

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $retry->webhook_url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $retry->payload,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => $this->buildHeaders($headers),
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode >= 400) {
            throw new \Exception("Webhook failed with HTTP {$httpCode}");
        }
    }

    private function buildHeaders(array $headers): array
    {
        $result = ['Content-Type: application/json'];
        foreach ($headers as $key => $value) {
            $result[] = "{$key}: {$value}";
        }
        return $result;
    }

    /**
     * Add jitter to delay to prevent thundering herd
     */
    private function randomizeDelay(int $baseDelay): string
    {
        $jitter = rand(0, $baseDelay * 0.1);
        $delay = $baseDelay + $jitter;

        return date('Y-m-d H:i:s', time() + $delay);
    }

    private function markFailed(array $webhookData): void
    {
        logActivity("Webhook permanently failed: {$webhookData['event_type']} - {$webhookData['url']}");
    }

    private function ensureRetryTableExists(): void
    {
        if (!Capsule::schema()->hasTable($this->retryTable)) {
            Capsule::schema()->create($this->retryTable, function($table) {
                $table->increments('id');
                $table->string('webhook_url', 500);
                $table->string('event_type', 100);
                $table->longText('payload');
                $table->text('headers');
                $table->integer('attempt')->default(1);
                $table->string('status', 20)->default('pending');
                $table->string('error_message', 500)->nullable();
                $table->timestamp('scheduled_at');
                $table->timestamp('created_at')->useCurrent();
                $table->timestamp('completed_at')->nullable();
                $table->index(['status', 'scheduled_at']);
            });
        }
    }
}

/**
 * Daily cron hook to process retries
 */
add_hook('DailyCronJob', 1, function() {
    $retryHandler = new WHMCS_Webhook_Retry_Handler();
    $processed = $retryHandler->processScheduledRetries();

    if ($processed > 0) {
        logActivity("Processed {$processed} webhook retries");
    }
});
```

### Step 5: Webhook Security Best Practices

Implement comprehensive security for webhook handling:

```php
// modules/addons/youraddon/includes/webhook_security.php

/**
 * Webhook Security Helper
 */
class WHMCS_Webhook_Security
{
    /**
     * IP Whitelist validation
     */
    public static function validateIPWhitelist(array $allowedIPs): bool
    {
        $clientIP = $_SERVER['REMOTE_ADDR'] ?? '';

        // Handle proxies
        if (isset($_SERVER['HTTP_X_FORWARDED_FOR'])) {
            $clientIP = $_SERVER['HTTP_X_FORWARDED_FOR'];
        }

        return in_array($clientIP, $allowedIPs);
    }

    /**
     * Timestamp validation to prevent replay attacks
     */
    public static function validateTimestamp(int $timestamp, int $tolerance = 300): bool
    {
        $diff = abs(time() - $timestamp);

        return $diff <= $tolerance;
    }

    /**
     * Request size validation
     */
    public static function validatePayloadSize(int $maxSize = 1048576): bool
    {
        $contentLength = $_SERVER['CONTENT_LENGTH'] ?? 0;

        return (int) $contentLength <= $maxSize;
    }

    /**
     * Content type validation
     */
    public static function validateContentType(string $allowedContentType): bool
    {
        $contentType = $_SERVER['CONTENT_TYPE'] ?? '';

        return strpos($contentType, $allowedContentType) !== false;
    }

    /**
     * Rate limit per IP
     */
    public static function checkIPRateLimit(string $ip, int $maxRequests = 100, int $window = 60): bool
    {
        $key = 'webhook_ip_' . md5($ip) . '_' . floor(time() / $window);
        $cacheKey = \Di::make('cache')->remember($key, $window, function() use ($maxRequests) {
            return Capsule::table('mod_webhook_ip_counts')
                ->where('ip_address', $ip)
                ->where('window_start', '>=', date('Y-m-d H:i:s', time() - $window))
                ->count();

            return $count;
        });

        if ($cacheKey >= $maxRequests) {
            return false;
        }

        Capsule::table('mod_webhook_ip_counts')->insertOnDuplicateKeyUpdate([
            'ip_address' => $ip,
            'window_start' => date('Y-m-d H:i:s', floor(time() / $window) * $window),
            'request_count' => 1,
        ], ['request_count' => Capsule::raw('request_count + 1')]);

        return true;
    }

    /**
     * Audit webhook access
     */
    public static function auditAccess(string $provider, string $ip, bool $allowed, string $reason = ''): void
    {
        Capsule::table('mod_webhook_audit_log')->insert([
            'provider' => $provider,
            'ip_address' => $ip,
            'allowed' => $allowed,
            'reason' => $reason,
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * Integrated security check
 */
class SecureWebhookHandler extends WebhookController
{
    private $allowedIPs = [];
    private $timestampTolerance = 300; // 5 minutes

    /**
     * Handle request with security checks
     */
    public function handle(): array
    {
        $ip = $_SERVER['REMOTE_ADDR'] ?? '';

        // Check IP rate limit
        if (!WHMCS_Webhook_Security::checkIPRateLimit($ip)) {
            WHMCS_Webhook_Security::auditAccess('unknown', $ip, false, 'Rate limit exceeded');
            return ['status' => 429, 'body' => ['error' => 'Too many requests']];
        }

        // Check payload size
        if (!WHMCS_Webhook_Security::validatePayloadSize(10 * 1024 * 1024)) {
            WHMCS_Webhook_Security::auditAccess('unknown', $ip, false, 'Payload too large');
            return ['status' => 413, 'body' => ['error' => 'Payload too large']];
        }

        // Validate IP whitelist if configured
        if (!empty($this->allowedIPs)) {
            if (!WHMCS_Webhook_Security::validateIPWhitelist($this->allowedIPs)) {
                WHMCS_Webhook_Security::auditAccess('unknown', $ip, false, 'IP not whitelisted');
                return ['status' => 403, 'body' => ['error' => 'Access denied']];
            }
        }

        return parent::handle();
    }
}
```

## Verification Checklist

```
Webhook Handler Setup:
□ Webhook endpoint registered
□ Signature verification implemented
□ Timestamp validation for replay protection
□ IP whitelist configured (if applicable)
□ Rate limiting enabled
□ Error logging functional
□ Retry logic configured
□ Audit logging enabled

Outbound Webhooks:
□ WHMCS events mapped to webhooks
□ SSL/TLS enabled for delivery
□ Signature headers included
□ Retry on failure configured
□ Delivery timeout set appropriately

Security:
□ HTTPS only for webhooks
□ Signature verification on all inbound webhooks
□ IP whitelisting for provider IPs
□ Rate limiting per IP
□ Audit logging for all attempts
□ Input validation on all received data
□ No sensitive data in URLs
```

## WHMCS Best Practices

1. **Always verify webhook signatures** before processing
2. **Use timing-safe comparison** for signature verification
3. **Implement timestamp validation** to prevent replay attacks
4. **Log all webhook events** for debugging and auditing
5. **Implement retry logic** with exponential backoff
6. **Return 200 quickly** and process asynchronously if needed
7. **Validate all input** before processing
8. **Use SSL/TLS** for all webhook communications
9. **Monitor delivery success rates** and alert on failures
10. **Implement idempotency** to handle duplicate webhook deliveries

## Common Pitfalls to Avoid

1. **Not validating signatures** - Always verify webhook authenticity
2. **Processing synchronously** - Return 200 quickly, process async
3. **Missing error handling** - Handle all failure scenarios
4. **Not logging failures** - Always log errors for debugging
5. **No retry logic** - Implement retry for failed deliveries
6. **Ignoring rate limits** - Respect external service limits
7. **Missing timeout handling** - Implement proper timeouts
8. **Not handling duplicates** - Use middleware for idempotency
9. **Hardcoding secrets** - Use secure configuration storage
10. **Missing audit trail** - Log all webhook access attempts

## WHMCS ClassDocs References

- [add_hook()](https://developers.whmcs.com/advanced/hooks-system/) - Hook system
- [logActivity()](https://developers.whmcs.com/advanced/logging/) - Activity logging
- [Capsule](https://developers.whmcs.com/pdo-wrapper/) - Database operations
- [encrypt()](https://developers.whmcs.com/advanced/encryption/) - Data encryption
