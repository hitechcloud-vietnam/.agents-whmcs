# WHMCS Webhook Automation Workflow

## Purpose

Set up and manage webhooks in WHMCS for real-time integration with external services, automated workflows, and event-driven architectures.

## Prerequisites

- WHMCS v7.2+ for native webhooks
- External service API access
- Webhook endpoint configuration
- SSL certificate for HTTPS endpoints

## Workflow Steps

### Step 1: Configure WHMCS Webhooks

Set up webhook system in WHMCS:

```php
<?php
// modules/custom/webhook_handlers.php

use WHMCS\WebHooks\Webhook;

// Register webhook handlers
add_hook('AfterInitialMalwareScan', 1, function($params) {
    $webhook = new Webhook('malware_scan_complete');
    $webhook->send($params);
});
```

### Step 2: Create Custom Webhook Handler

Build custom webhook endpoints:

```php
<?php
// modules/custom/api/WebhookController.php

namespace WHMCS\Custom\Api;

class WebhookController extends ApiHandler
{
    private $validSignatures = [];
    private $webhookSecret;
    
    public function __construct()
    {
        $this->webhookSecret = $_ENV['WEBHOOK_SECRET'] ?? '';
    }
    
    public function handleTicketCreated()
    {
        try {
            $payload = $this->getPayload();
            $this->verifySignature();
            
            $ticketId = $payload['ticket_id'];
            $clientId = $payload['client_id'];
            $subject = $payload['subject'];
            
            logActivity("Webhook: Ticket created - {$ticketId}");
            
            // Process ticket creation
            $result = $this->processTicketCreation($ticketId, $payload);
            
            return $this->success($result)->send();
            
        } catch (\Exception $e) {
            return $this->error($e->getMessage(), 400)->send();
        }
    }
    
    public function handleInvoicePaid()
    {
        try {
            $payload = $this->getPayload();
            $this->verifySignature();
            
            $invoiceId = $payload['invoice_id'];
            $amount = $payload['amount'];
            
            logActivity("Webhook: Invoice paid - {$invoiceId} - {$amount}");
            
            // Trigger downstream actions
            $this->processInvoicePayment($invoiceId, $payload);
            
            return $this->success(['processed' => true])->send();
            
        } catch (\Exception $e) {
            return $this->error($e->getMessage(), 400)->send();
        }
    }
    
    public function handleServiceCreated()
    {
        try {
            $payload = $this->getPayload();
            $this->verifySignature();
            
            $serviceId = $payload['service_id'];
            $domain = $payload['domain'];
            
            logActivity("Webhook: Service created - {$serviceId}");
            
            // Provision related resources
            $this->provisionRelatedResources($serviceId, $payload);
            
            return $this->success(['provisioned' => true])->send();
            
        } catch (\Exception $e) {
            return $this->error($e->getMessage(), 400)->send();
        }
    }
    
    private function getPayload()
    {
        $rawPayload = file_get_contents('php://input');
        
        if (empty($rawPayload)) {
            throw new \Exception('Empty payload');
        }
        
        $payload = json_decode($rawPayload, true);
        
        if (json_last_error() !== JSON_ERROR_NONE) {
            throw new \Exception('Invalid JSON payload');
        }
        
        return $payload;
    }
    
    private function verifySignature()
    {
        $signature = $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? '';
        
        if (empty($signature)) {
            throw new \Exception('Missing webhook signature');
        }
        
        $expectedSignature = hash_hmac('sha256', file_get_contents('php://input'), $this->webhookSecret);
        
        if (!hash_equals($expectedSignature, $signature)) {
            throw new \Exception('Invalid webhook signature');
        }
    }
    
    private function processTicketCreation($ticketId, $payload)
    {
        // Send to external helpdesk system
        $externalSystem = new ExternalHelpdeskAPI();
        
        $result = $externalSystem->createTicket([
            'external_id' => $ticketId,
            'subject' => $payload['subject'],
            'description' => $payload['message'],
            'priority' => $this->mapPriority($payload['priority']),
            'client_email' => $payload['client_email'],
        ]);
        
        return $result;
    }
    
    private function processInvoicePayment($invoiceId, $payload)
    {
        // Trigger fulfillment if needed
        $invoice = Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->first();
        
        if ($invoice->status === 'Paid') {
            // Trigger automatic provisioning for new orders
            $this->fulfillPendingOrders($invoiceId);
            
            // Send confirmation to external accounting system
            $this->syncToAccounting($invoice);
        }
    }
    
    private function provisionRelatedResources($serviceId, $payload)
    {
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();
        
        // Create monitoring setup
        $monitoring = new MonitoringService();
        $monitoring->addHost($service->domain, [
            'service_type' => 'https',
            'check_interval' => 60,
            'alert_threshold' => 3,
        ]);
        
        // Set up backup automation
        $backup = new BackupService();
        $backup->configureBackup($service->id);
    }
}
```

### Step 3: Create Outgoing Webhooks

Send events to external systems:

```php
<?php
// modules/custom/outgoing_webhooks.php

namespace WHMCS\Custom;

class WebhookDispatcher
{
    private $endpoints = [];
    private $retryQueue = [];
    private $maxRetries = 3;
    
    public function __construct()
    {
        $this->loadEndpoints();
    }
    
    private function loadEndpoints()
    {
        $this->endpoints = Capsule::table('mod_webhook_endpoints')
            ->where('enabled', 1)
            ->get()
            ->groupBy('event_type')
            ->toArray();
    }
    
    public function dispatch($eventType, $payload)
    {
        if (!isset($this->endpoints[$eventType])) {
            return;
        }
        
        foreach ($this->endpoints[$eventType] as $endpoint) {
            $this->sendToEndpoint($endpoint, $eventType, $payload);
        }
    }
    
    private function sendToEndpoint($endpoint, $eventType, $payload)
    {
        $data = [
            'event' => $eventType,
            'timestamp' => date('c'),
            'data' => $payload,
        ];
        
        $headers = [
            'Content-Type: application/json',
            'X-Webhook-Event: ' . $eventType,
            'X-Webhook-Signature: ' . $this->generateSignature($data),
        ];
        
        $ch = curl_init($endpoint->url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_SSL_VERIFYPEER => true,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        // Log the webhook attempt
        $this->logWebhook($endpoint->id, $eventType, $httpCode, $response);
        
        // Retry on failure
        if ($httpCode < 200 || $httpCode >= 300) {
            $this->queueForRetry($endpoint, $eventType, $payload);
        }
    }
    
    private function generateSignature($data)
    {
        $secret = $_ENV['WEBHOOK_SIGNING_SECRET'] ?? '';
        return hash_hmac('sha256', json_encode($data), $secret);
    }
    
    private function logWebhook($endpointId, $eventType, $httpCode, $response)
    {
        Capsule::table('mod_webhook_logs')->insert([
            'endpoint_id' => $endpointId,
            'event_type' => $eventType,
            'http_code' => $httpCode,
            'response' => substr($response, 0, 1000),
            'sent_at' => Carbon::now()->toDateTimeString(),
        ]);
    }
    
    private function queueForRetry($endpoint, $eventType, $payload)
    {
        Capsule::table('mod_webhook_retry_queue')->insert([
            'endpoint_id' => $endpoint->id,
            'endpoint_url' => $endpoint->url,
            'event_type' => $eventType,
            'payload' => json_encode($payload),
            'attempts' => 0,
            'next_retry' => Carbon::now()->addMinutes(5)->toDateTimeString(),
        ]);
    }
}

// Hook: Dispatch webhook on events
add_hook('AfterInvoicePaymentApplication', 1, function($vars) {
    $dispatcher = new \WHMCS\Custom\WebhookDispatcher();
    $dispatcher->dispatch('invoice.paid', [
        'invoice_id' => $vars['invoiceid'],
        'amount' => $vars['amount'],
        'client_id' => $vars['userid'],
    ]);
});

add_hook('AfterServiceCreate', 1, function($vars) {
    $dispatcher = new \WHMCS\Custom\WebhookDispatcher();
    $dispatcher->dispatch('service.created', [
        'service_id' => $vars['serviceId'],
        'domain' => $vars['params']['domain'],
        'client_id' => $vars['params']['clientId'],
    ]);
});
```

### Step 4: Create Webhook Retry Mechanism

Handle failed webhook deliveries:

```php
<?php
// modules/custom/webhook_retry_cron.php
// Run every 5 minutes via cron

require_once __DIR__ . '/init.php';

class WebhookRetryProcessor
{
    private $dispatcher;
    
    public function __construct()
    {
        $this->dispatcher = new \WHMCS\Custom\WebhookDispatcher();
    }
    
    public function processRetries()
    {
        $pendingRetries = $this->getPendingRetries();
        
        foreach ($pendingRetries as $retry) {
            $this->processRetry($retry);
        }
    }
    
    private function getPendingRetries()
    {
        return Capsule::table('mod_webhook_retry_queue')
            ->where('next_retry', '<=', Carbon::now()->toDateTimeString())
            ->where('attempts', '<', 5)
            ->get();
    }
    
    private function processRetry($retry)
    {
        // Increment attempt count
        Capsule::table('mod_webhook_retry_queue')
            ->where('id', $retry->id)
            ->increment('attempts');
        
        // Attempt delivery
        $result = $this->deliverWebhook($retry);
        
        if ($result['success']) {
            // Remove from retry queue
            Capsule::table('mod_webhook_retry_queue')
                ->where('id', $retry->id)
                ->delete();
                
            logActivity("Webhook retry successful for {$retry->endpoint_url}");
        } else {
            // Schedule next retry with exponential backoff
            $nextRetry = Carbon::now()->addMinutes(pow(2, $retry->attempts));
            
            Capsule::table('mod_webhook_retry_queue')
                ->where('id', $retry->id)
                ->update([
                    'next_retry' => $nextRetry->toDateTimeString(),
                    'last_error' => $result['error'],
                ]);
        }
    }
    
    private function deliverWebhook($retry)
    {
        $data = [
            'event' => $retry->event_type,
            'timestamp' => date('c'),
            'data' => json_decode($retry->payload, true),
            'retry_attempt' => $retry->attempts,
        ];
        
        $headers = [
            'Content-Type: application/json',
            'X-Webhook-Event: ' . $retry->event_type,
            'X-Webhook-Retry: ' . $retry->attempts,
        ];
        
        $ch = curl_init($retry->endpoint_url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode >= 200 && $httpCode < 300) {
            return ['success' => true];
        }
        
        return [
            'success' => false,
            'error' => "HTTP {$httpCode}: " . substr($response, 0, 200),
        ];
    }
}

// Process retries
$processor = new WebhookRetryProcessor();
$processor->processRetries();
```

### Step 5: Create Webhook Management Interface

Admin dashboard for webhook management:

```php
<?php
// admin/webhooks.php

require_once __DIR__ . '/../init.php';

if (!checkPermission('Configure Webhooks', true)) {
    exit('Access Denied');
}

$action = $_GET['action'] ?? 'list';

switch ($action) {
    case 'list':
        echo $twig->render('admin/webhooks/list.html', [
            'endpoints' => getWebhookEndpoints(),
            'stats' => getWebhookStats(),
            'recent_logs' => getRecentLogs(50),
        ]);
        break;
        
    case 'create':
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            createWebhookEndpoint($_POST);
            redir('action=list&success=created');
        }
        echo $twig->render('admin/webhooks/create.html', [
            'event_types' => getAvailableEventTypes(),
        ]);
        break;
        
    case 'test':
        $endpointId = (int) $_GET['id'];
        testWebhookEndpoint($endpointId);
        redir('action=list&success=test_sent');
        break;
        
    case 'logs':
        $endpointId = (int) ($_GET['id'] ?? 0);
        echo $twig->render('admin/webhooks/logs.html', [
            'endpoint' => getEndpoint($endpointId),
            'logs' => getEndpointLogs($endpointId, 100),
        ]);
        break;
}

function getWebhookEndpoints()
{
    return Capsule::table('mod_webhook_endpoints')
        ->orderBy('created_at', 'desc')
        ->get();
}

function getWebhookStats()
{
    return [
        'total' => Capsule::table('mod_webhook_endpoints')->count(),
        'enabled' => Capsule::table('mod_webhook_endpoints')->where('enabled', 1)->count(),
        'sent_today' => Capsule::table('mod_webhook_logs')
            ->where('sent_at', '>=', Carbon::today()->toDateTimeString())
            ->count(),
        'failed_today' => Capsule::table('mod_webhook_logs')
            ->where('sent_at', '>=', Carbon::today()->toDateTimeString())
            ->where('http_code', '>=', 400)
            ->count(),
    ];
}

function getAvailableEventTypes()
{
    return [
        'invoice.paid' => 'Invoice Paid',
        'invoice.created' => 'Invoice Created',
        'service.created' => 'Service Created',
        'service.suspended' => 'Service Suspended',
        'service.terminated' => 'Service Terminated',
        'ticket.created' => 'Ticket Created',
        'ticket.reply' => 'Ticket Reply',
        'client.created' => 'Client Created',
        'domain.registered' => 'Domain Registered',
        'domain.transfer_completed' => 'Domain Transfer Completed',
    ];
}
```

### Step 6: Register Webhooks with External Services

Connect WHMCS to external platforms:

```php
<?php
// modules/custom/connectors/slack_connector.php

namespace WHMCS\Custom\Connectors;

class SlackConnector
{
    private $webhookUrl;
    private $channel;
    
    public function __construct($webhookUrl, $channel = '#whmcs-alerts')
    {
        $this->webhookUrl = $webhookUrl;
        $this->channel = $channel;
    }
    
    public function sendNotification($event, $data)
    {
        $message = $this->formatMessage($event, $data);
        
        $payload = [
            'channel' => $this->channel,
            'username' => 'WHMCS Bot',
            'icon_emoji' => ':robot_face:',
            'attachments' => [$message],
        ];
        
        return $this->send($payload);
    }
    
    private function formatMessage($event, $data)
    {
        $colors = [
            'invoice.paid' => '#36a64f',
            'service.created' => '#0074d9',
            'service.suspended' => '#ff851b',
            'service.terminated' => '#ff4136',
        ];
        
        $titles = [
            'invoice.paid' => 'Payment Received',
            'service.created' => 'New Service',
            'service.suspended' => 'Service Suspended',
            'service.terminated' => 'Service Terminated',
        ];
        
        return [
            'color' => $colors[$event] ?? '#aaaaaa',
            'title' => $titles[$event] ?? $event,
            'fields' => $this->formatFields($event, $data),
            'footer' => 'WHMCS Webhook',
            'ts' => time(),
        ];
    }
    
    private function formatFields($event, $data)
    {
        $fields = [];
        
        switch ($event) {
            case 'invoice.paid':
                $fields[] = ['title' => 'Invoice #', 'value' => $data['invoice_id'], 'short' => true];
                $fields[] = ['title' => 'Amount', 'value' => $data['amount'], 'short' => true];
                break;
            case 'service.created':
                $fields[] = ['title' => 'Domain', 'value' => $data['domain'], 'short' => true];
                $fields[] = ['title' => 'Product', 'value' => $data['product_name'] ?? 'N/A', 'short' => true];
                break;
        }
        
        return $fields;
    }
    
    private function send($payload)
    {
        $ch = curl_init($this->webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $result = curl_exec($ch);
        curl_close($ch);
        
        return $result === 'ok';
    }
}
```

## Verification Checklist

- [ ] Webhook endpoints configured
- [ ] Signature verification implemented
- [ ] Incoming webhook handlers created
- [ ] Outgoing webhook dispatcher working
- [ ] Retry mechanism configured
- [ ] Admin dashboard accessible
- [ ] Logging enabled
- [ ] External integrations connected
- [ ] Testing completed
- [ ] Monitoring in place

## Related Skills and Documentation

- [WHMCS API Development](whmcs-api-development-workflow.md)
- [WHMCS Integration Testing](whmcs-integration-testing-workflow.md)
- WHMCS Webhooks Documentation: https://developers.whmcs.com/advanced/webhooks/
- Webhook Security Best Practices: https://developers.whmcs.com/authentication/webhooks/

## Notes

- Always verify webhook signatures
- Use HTTPS for all webhook endpoints
- Implement retry logic for reliability
- Log all webhook activity for debugging
- Monitor webhook delivery success rates
- Keep webhook secrets secure
- Test webhooks in development first
- Consider idempotency for duplicate events
