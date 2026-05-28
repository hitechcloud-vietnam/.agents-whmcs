# WHMCS Webhook Integration Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Comprehensive guide to implementing webhook integrations in WHMCS for real-time event notifications, external system synchronization, and automated workflow triggers.

## Prerequisites

- WHMCS installation with API access enabled
- External API endpoints for receiving webhooks
- SSL certificate for secure webhook delivery
- Webhook receiver endpoint capable of handling HTTP POST

## Workflow Steps

### Step 1: Understand WHMCS Webhook Events

Identify available webhook trigger points:

```php
// Key WHMCS hooks that can trigger webhooks
$webhookEvents = [
    'Client' => [
        'ClientAdd'           => 'After new client registration',
        'ClientLogin'         => 'Client login event',
        'ClientLogout'        => 'Client logout event',
        'ClientChangePassword'=> 'Password changed',
        'ClientUpdate'        => 'Client profile updated',
    ],
    'Order' => [
        'OrderCreated'        => 'New order placed',
        'OrderPaid'          => 'Order payment confirmed',
        'OrderCancelled'      => 'Order cancelled',
        'OrderRefunded'       => 'Order refunded',
    ],
    'Invoice' => [
        'InvoiceCreation'     => 'Invoice created',
        'InvoicePaid'         => 'Invoice payment received',
        'InvoiceOverdue'      => 'Invoice becomes overdue',
        'InvoiceCancelled'   => 'Invoice cancelled',
    ],
    'Service' => [
        'AfterModuleCreate'   => 'Service provisioned',
        'AfterModuleSuspend'  => 'Service suspended',
        'AfterModuleUnsuspend'=> 'Service reactivated',
        'AfterModuleTerminate'=> 'Service terminated',
        'AfterModuleChangePassword'=> 'Password changed',
    ],
    'Domain' => [
        'DomainRegistration'  => 'Domain registered',
        'DomainTransfer'      => 'Domain transfer completed',
        'DomainRenewal'       => 'Domain renewed',
        'DomainDeletion'     => 'Domain deleted',
    ],
    'Support' => [
        'TicketOpen'         => 'Support ticket opened',
        'TicketAdminReply'   => 'Admin replied to ticket',
        'TicketUserReply'    => 'User replied to ticket',
        'TicketClose'        => 'Ticket closed',
    ],
];
```

### Step 2: Create Webhook Manager Class

Build core webhook management functionality:

```php
// includes/classes/WebhookManager.php

use WHMCS\Database\Capsule;

class WebhookManager
{
    private string $webhookUrl;
    private string $apiKey;
    private int $timeout = 30;
    private int $retryAttempts = 3;
    private int $retryDelay = 5;

    public function __construct(string $webhookUrl, string $apiKey = null)
    {
        $this->webhookUrl = $webhookUrl;
        $this->apiKey = $apiKey ?? $this->getDefaultApiKey();
    }

    /**
     * Send webhook notification
     */
    public function send(string $event, array $data): array
    {
        $payload = $this->buildPayload($event, $data);

        $attempt = 0;
        $lastError = null;

        while ($attempt < $this->retryAttempts) {
            $attempt++;

            try {
                $response = $this->makeRequest($payload);

                // Log successful delivery
                $this->logWebhook($event, $payload, $response, true);

                return [
                    'success' => true,
                    'response' => $response,
                    'attempt' => $attempt,
                ];
            } catch (Exception $e) {
                $lastError = $e->getMessage();

                if ($attempt < $this->retryAttempts) {
                    sleep($this->retryDelay);
                }
            }
        }

        // Log failed delivery
        $this->logWebhook($event, $payload, $lastError, false);

        return [
            'success' => false,
            'error' => $lastError,
            'attempts' => $attempt,
        ];
    }

    private function buildPayload(string $event, array $data): array
    {
        return [
            'event' => $event,
            'timestamp' => date('c'),
            'whmcs_version' => Capsule::table('tblconfiguration')
                ->where('setting', 'Version')->first()->value ?? 'Unknown',
            'data' => $data,
        ];
    }

    private function makeRequest(array $payload): array
    {
        $jsonPayload = json_encode($payload);

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->webhookUrl,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $jsonPayload,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'Authorization: Bearer ' . $this->apiKey,
                'X-Webhook-Signature: ' . $this->generateSignature($jsonPayload),
                'X-Webhook-Event: ' . $payload['event'],
            ],
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode < 200 || $httpCode >= 300) {
            throw new Exception("HTTP {$httpCode}: {$response}");
        }

        return json_decode($response, true) ?? ['status' => 'ok'];
    }

    private function generateSignature(string $payload): string
    {
        return hash_hmac('sha256', $payload, $this->apiKey);
    }

    private function getDefaultApiKey(): string
    {
        return Capsule::table('tblconfiguration')
            ->where('setting', 'WebhookApiKey')
            ->first()->value ?? '';
    }

    private function logWebhook(string $event, array $payload, $response, bool $success): void
    {
        Capsule::table('mod_webhook_logs')->insert([
            'event' => $event,
            'payload' => json_encode($payload),
            'response' => is_array($response) ? json_encode($response) : $response,
            'success' => $success ? 1 : 0,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

### Step 3: Implement Event Webhook Hooks

Create hooks to trigger webhooks on WHMCS events:

```php
// includes/hooks/webhook_events.php

/**
 * Client registration webhook
 */
add_hook('ClientAdd', 1, function(array $vars) {
    $webhook = new WebhookManager(
        'https://api.example.com/webhooks/whmcs',
        'your_api_key'
    );

    $webhook->send('client.registered', [
        'client_id'   => $vars['user_id'],
        'email'       => $vars['email'],
        'first_name'  => $vars['firstname'],
        'last_name'   => $vars['lastname'],
        'company'     => $vars['companyname'] ?? '',
    ]);
});

/**
 * Order paid webhook
 */
add_hook('OrderPaid', 1, function(array $vars) {
    $webhook = new WebhookManager(
        'https://api.example.com/webhooks/whmcs',
        'your_api_key'
    );

    $order = Capsule::table('tblorders')
        ->where('id', $vars['order_id'])
        ->first();

    $webhook->send('order.paid', [
        'order_id'    => $vars['order_id'],
        'order_number'=> $order->ordernum,
        'user_id'     => $order->userid,
        'total'       => $order->totaldue,
        'items'       => $this->getOrderItems($vars['order_id']),
    ]);
});

/**
 * Invoice paid webhook
 */
add_hook('InvoicePaid', 1, function(array $vars) {
    $webhook = new WebhookManager(
        'https://api.example.com/webhooks/whmcs',
        'your_api_key'
    );

    $invoice = Capsule::table('tblinvoices')
        ->where('id', $vars['invoice_id'])
        ->first();

    $webhook->send('invoice.paid', [
        'invoice_id'  => $vars['invoice_id'],
        'user_id'     => $invoice->userid,
        'amount'      => $invoice->total,
        'subtotal'    => $invoice->subtotal,
        'tax'         => $invoice->tax,
        'payment_method' => $invoice->paymentmethod,
    ]);
});

/**
 * Service provisioning webhook
 */
add_hook('AfterModuleCreate', 1, function(array $vars) {
    $webhook = new WebhookManager(
        'https://api.example.com/webhooks/whmcs',
        'your_api_key'
    );

    $service = Capsule::table('tblhosting')
        ->where('id', $vars['serviceid'])
        ->first();

    $product = Capsule::table('tblproducts')
        ->where('id', $service->packageid)
        ->first();

    $webhook->send('service.provisioned', [
        'service_id'  => $vars['serviceid'],
        'user_id'     => $service->userid,
        'domain'      => $service->domain,
        'product'     => $product->name,
        'server'      => Capsule::table('tblservers')
            ->where('id', $service->server)->first()->name ?? 'N/A',
    ]);
});

/**
 * Service suspend webhook
 */
add_hook('AfterModuleSuspend', 1, function(array $vars) {
    $webhook = new WebhookManager(
        'https://api.example.com/webhooks/whmcs',
        'your_api_key'
    );

    $webhook->send('service.suspended', [
        'service_id' => $vars['serviceid'],
        'reason'    => $_POST['suspendreason'] ?? 'Payment overdue',
    ]);
});

/**
 * Ticket open webhook
 */
add_hook('TicketOpen', 1, function(array $vars) {
    $webhook = new WebhookManager(
        'https://api.example.com/webhooks/whmcs',
        'your_api_key'
    );

    $ticket = Capsule::table('tbltickets')
        ->where('id', $vars['ticket_id'])
        ->first();

    $webhook->send('ticket.opened', [
        'ticket_id'   => $vars['ticket_id'],
        'ticket_number'=> $ticket->ticketnum,
        'subject'     => $ticket->subject,
        'priority'    => $ticket->urgency,
        'user_id'     => $ticket->userid,
    ]);
});
```

### Step 4: Create Webhook Configuration Admin Panel

Build admin interface for webhook management:

```php
// modules/addons/webhook_manager/webhook_manager.php

function webhook_manager_config(): array
{
    return [
        'name'        => 'Webhook Manager',
        'description' => 'Configure and manage webhooks',
        'version'     => '1.0',
    ];
}

function webhook_manager_activate(): array
{
    Capsule::schema()->create('mod_webhook_configs', function($t) {
        $t->increments('id');
        $t->string('name');
        $t->string('url');
        $t->string('api_key')->nullable();
        $t->text('events'); // JSON array of events
        $t->enum('status', ['active', 'inactive']);
        $t->integer('retry_count')->default(3);
        $t->integer('timeout')->default(30);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_webhook_logs', function($t) {
        $t->increments('id');
        $t->string('webhook_id');
        $t->string('event');
        $t->text('payload');
        $t->text('response');
        $t->boolean('success');
        $t->timestamp('created_at')->useCurrent();
    });

    return ['status' => 'success'];
}

function webhook_manager_output(array $vars): void
{
    check_token('WHMCS.admin.default');

    $action = $_REQUEST['subaction'] ?? 'list';

    switch ($action) {
        case 'list':
            $webhooks = Capsule::table('mod_webhook_configs')->get();
            include __DIR__ . '/views/list.php';
            break;

        case 'add':
            include __DIR__ . '/views/add.php';
            break;

        case 'edit':
            $webhook = Capsule::table('mod_webhook_configs')
                ->where('id', $_REQUEST['id'])
                ->first();
            include __DIR__ . '/views/edit.php';
            break;

        case 'logs':
            $logs = Capsule::table('mod_webhook_logs')
                ->where('webhook_id', $_REQUEST['id'])
                ->orderBy('created_at', 'DESC')
                ->limit(100)
                ->get();
            include __DIR__ . '/views/logs.php';
            break;

        case 'save':
            $this->saveWebhook($_POST);
            redir('module=webhook_manager');
            break;

        case 'delete':
            Capsule::table('mod_webhook_configs')
                ->where('id', $_REQUEST['id'])
                ->delete();
            redir('module=webhook_manager');
            break;

        case 'test':
            $result = $this->testWebhook($_REQUEST['id']);
            echo json_encode($result);
            break;
    }
}
```

### Step 5: Implement Webhook Security

Add robust security measures:

```php
// includes/classes/WebhookSecurity.php

class WebhookSecurity
{
    /**
     * Verify webhook signature
     */
    public function verifySignature(string $payload, string $signature, string $secret): bool
    {
        $expected = hash_hmac('sha256', $payload, $secret);
        return hash_equals($expected, $signature);
    }

    /**
     * Verify webhook IP whitelist
     */
    public function verifyIpWhitelist(string $clientIp, array $allowedIps): bool
    {
        if (empty($allowedIps)) {
            return true; // No whitelist configured
        }

        return in_array($clientIp, $allowedIps);
    }

    /**
     * Validate incoming webhook payload
     */
    public function validatePayload(array $payload, string $signature = null): bool
    {
        $requiredFields = ['event', 'timestamp', 'data'];

        foreach ($requiredFields as $field) {
            if (!isset($payload[$field])) {
                logActivity('Webhook validation failed: missing ' . $field);
                return false;
            }
        }

        // Verify signature if provided
        if ($signature) {
            $jsonPayload = json_encode($payload);
            $webhookSecret = $this->getWebhookSecret();

            if (!$this->verifySignature($jsonPayload, $signature, $webhookSecret)) {
                logActivity('Webhook signature verification failed');
                return false;
            }
        }

        return true;
    }

    /**
     * Rate limiting
     */
    private array $rateLimits = [];

    public function checkRateLimit(string $webhookId, int $maxRequests = 100): bool
    {
        $key = 'webhook_rate_' . $webhookId;
        $current = (int) $_SESSION[$key] ?? 0;
        $lastReset = (int) $_SESSION[$key . '_reset'] ?? 0;

        // Reset counter if hour has passed
        if (time() - $lastReset > 3600) {
            $current = 0;
            $lastReset = time();
            $_SESSION[$key . '_reset'] = $lastReset;
        }

        if ($current >= $maxRequests) {
            logActivity("Webhook rate limit exceeded: {$webhookId}");
            return false;
        }

        $_SESSION[$key] = $current + 1;
        return true;
    }
}
```

---

## Best Practices

1. **Always use HTTPS** - Transmit webhooks over encrypted connections
2. **Sign all payloads** - Use HMAC signatures for verification
3. **Implement idempotency** - Handle duplicate webhook deliveries gracefully
4. **Log all deliveries** - Maintain records of sent and received webhooks
5. **Implement retries** - Use exponential backoff for failed deliveries
6. **Validate all input** - Check payload structure before processing
7. **Use queues** - Process webhooks asynchronously to avoid delays
8. **Monitor delivery rates** - Track success and failure metrics
9. **Set timeouts appropriately** - Balance responsiveness with reliability
10. **Provide status endpoint** - Allow recipients to check webhook status

---

## Verification Checklist

- [ ] Webhook configuration panel accessible in admin
- [ ] Events correctly trigger webhooks on WHMCS events
- [ ] Payloads include all required data fields
- [ ] Signature verification working correctly
- [ ] Failed webhooks retried as configured
- [ ] Webhook logs recording all deliveries
- [ ] Rate limiting enforced appropriately
- [ ] IP whitelist verification active
- [ ] Test webhook sends successfully
- [ ] Admin can enable/disable webhooks per event
