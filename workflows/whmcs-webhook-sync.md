# WHMCS Webhook Synchronization Workflow

## Purpose
Implement webhook-based real-time synchronization for WHMCS events.

## Prerequisites
- WHMCS installation
- Webhook receiver endpoint
- SSL certificate

## Step-by-Step Process

### Step 1: Create Webhook Manager

**Create hooks/webhook_sync.php:**
```php
<?php
/**
 * WHMCS Webhook Synchronization Handler
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class WebhookSync {
    
    private $secret;
    private $webhookUrl;
    private $eventQueue = [];
    
    public function __construct($config = []) {
        $this->secret = $config['secret'] ?? '';
        $this->webhookUrl = $config['webhook_url'] ?? '';
    }
    
    /**
     * Generate webhook signature
     */
    public function generateSignature($payload) {
        return hash_hmac('sha256', $payload, $this->secret);
    }
    
    /**
     * Verify webhook signature
     */
    public function verifySignature($payload, $signature) {
        $expected = $this->generateSignature($payload);
        return hash_equals($expected, $signature);
    }
    
    /**
     * Send webhook event
     */
    public function sendWebhook($event, $data) {
        if (empty($this->webhookUrl)) {
            return ['success' => false, 'error' => 'Webhook URL not configured'];
        }
        
        $payload = json_encode([
            'event' => $event,
            'timestamp' => Carbon::now()->toDateTimeString(),
            'data' => $data
        ]);
        
        $signature = $this->generateSignature($payload);
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->webhookUrl,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-Webhook-Signature: ' . $signature,
                'X-Webhook-Event: ' . $event
            ],
            CURLOPT_TIMEOUT => 30
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        $success = $httpCode >= 200 && $httpCode < 300;
        
        // Log webhook
        $this->logWebhook($event, $data, $httpCode, $response);
        
        return [
            'success' => $success,
            'http_code' => $httpCode,
            'response' => $response
        ];
    }
    
    /**
     * Queue webhook for later sending
     */
    public function queueWebhook($event, $data) {
        $this->eventQueue[] = [
            'event' => $event,
            'data' => $data,
            'queued_at' => Carbon::now()->toDateTimeString()
        ];
    }
    
    /**
     * Process queued webhooks
     */
    public function processQueue() {
        $results = [];
        
        foreach ($this->eventQueue as $index => $webhook) {
            $result = $this->sendWebhook($webhook['event'], $webhook['data']);
            $results[] = $result;
            
            if ($result['success']) {
                unset($this->eventQueue[$index]);
            }
        }
        
        // Save remaining queue
        $this->saveQueue();
        
        return $results;
    }
    
    /**
     * Log webhook event
     */
    private function logWebhook($event, $data, $httpCode, $response) {
        $table = 'mod_webhook_logs';
        
        if (!Capsule::schema()->hasTable($table)) {
            Capsule::schema()->create($table, function($t) {
                $t->increments('id');
                $t->string('event', 100);
                $t->json('data');
                $t->integer('http_code');
                $t->text('response')->nullable();
                $t->timestamp('created_at');
            });
        }
        
        Capsule::table($table)->insert([
            'event' => $event,
            'data' => json_encode($data),
            'http_code' => $httpCode,
            'response' => substr($response, 0, 1000),
            'created_at' => Carbon::now()->toDateTimeString()
        ]);
    }
    
    /**
     * Save queue to file
     */
    private function saveQueue() {
        $queueFile = ROOTDIR . '/data/webhook_queue.json';
        file_put_contents($queueFile, json_encode($this->eventQueue));
    }
    
    /**
     * Load queue from file
     */
    public function loadQueue() {
        $queueFile = ROOTDIR . '/data/webhook_queue.json';
        if (file_exists($queueFile)) {
            $this->eventQueue = json_decode(file_get_contents($queueFile), true) ?? [];
        }
    }
}

// Initialize webhook sync
$webhookSync = new WebhookSync([
    'secret' => \WHMCS\Config\Setting::getValue('WebhookSecret'),
    'webhook_url' => \WHMCS\Config\Setting::getValue('WebhookUrl')
]);
```

### Step 2: Register Webhook Hooks

```php
<?php
/**
 * Register webhook event handlers
 */

add_hook('ClientAdd', 1, function($vars) use ($webhookSync) {
    $webhookSync->sendWebhook('client.created', [
        'id' => $vars['userid'],
        'email' => $vars['email'],
        'name' => $vars['firstname'] . ' ' . $vars['lastname']
    ]);
});

add_hook('ClientEdit', 1, function($vars) use ($webhookSync) {
    $webhookSync->sendWebhook('client.updated', [
        'id' => $vars['userid'],
        'changes' => $vars
    ]);
});

add_hook('AfterModuleCreate', 1, function($vars) use ($webhookSync) {
    $webhookSync->sendWebhook('service.created', [
        'id' => $vars['serviceid'],
        'client_id' => $vars['userid'],
        'product_id' => $vars['productId'],
        'domain' => $vars['domain'] ?? ''
    ]);
});

add_hook('InvoicePaid', 1, function($vars) use ($webhookSync) {
    $webhookSync->sendWebhook('invoice.paid', [
        'invoice_id' => $vars['invoice_id'],
        'client_id' => $vars['userid'],
        'amount' => $vars['amount'],
        'payment_method' => $vars['payment_method'] ?? 'unknown'
    ]);
});

add_hook('TicketOpen', 1, function($vars) use ($webhookSync) {
    $webhookSync->sendWebhook('ticket.created', [
        'ticket_id' => $vars['ticket_id'],
        'client_id' => $vars['userid'],
        'subject' => $vars['subject'],
        'department' => $vars['department']
    ]);
});

add_hook('DomainRegister', 1, function($vars) use ($webhookSync) {
    $webhookSync->sendWebhook('domain.registered', [
        'domain_id' => $vars['domainid'],
        'domain' => $vars['domain'],
        'registrar' => $vars['registrar']
    ]);
});

add_hook('DomainTransferComplete', 1, function($vars) use ($webhookSync) {
    $webhookSync->sendWebhook('domain.transferred', [
        'domain_id' => $vars['domainid'],
        'domain' => $vars['domain']
    ]);
});
```

### Step 3: Create Webhook Receiver Endpoint

**Create /includes/webhook_receiver.php:**
```php
<?php
/**
 * Webhook Receiver Endpoint
 * Handles incoming webhooks from WHMCS
 */

require_once __DIR__ . '/init.php';

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

// Verify request
if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    exit('Method Not Allowed');
}

// Get payload
$payload = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? '';
$event = $_SERVER['HTTP_X_WEBHOOK_EVENT'] ?? '';

// Verify signature
$secret = \WHMCS\Config\Setting::getValue('WebhookSecret');
$expectedSignature = hash_hmac('sha256', $payload, $secret);

if (!hash_equals($expectedSignature, $signature)) {
    http_response_code(401);
    exit('Unauthorized');
}

// Parse payload
$data = json_decode($payload, true);

if (!$data) {
    http_response_code(400);
    exit('Invalid JSON');
}

// Process webhook
$results = [
    'received_at' => Carbon::now()->toDateTimeString(),
    'event' => $event,
    'status' => 'processed'
];

switch ($event) {
    case 'client.created':
        $results['action'] = handleClientCreated($data);
        break;
        
    case 'service.created':
        $results['action'] = handleServiceCreated($data);
        break;
        
    case 'invoice.paid':
        $results['action'] = handleInvoicePaid($data);
        break;
        
    case 'ticket.created':
        $results['action'] = handleTicketCreated($data);
        break;
        
    default:
        $results['status'] = 'unknown_event';
}

// Log webhook
logWebhook($event, $data, $results);

http_response_code(200);
header('Content-Type: application/json');
echo json_encode($results);

/**
 * Handle client created event
 */
function handleClientCreated($data) {
    // Sync to external system
    // Send welcome email
    // Create account in other systems
    return 'client_processed';
}

/**
 * Handle service created event
 */
function handleServiceCreated($data) {
    // Provision external service
    // Send welcome email
    // Create related accounts
    return 'service_processed';
}

/**
 * Handle invoice paid event
 */
function handleInvoicePaid($data) {
    // Activate service if needed
    // Update external billing
    // Send confirmation
    return 'invoice_processed';
}

/**
 * Handle ticket created event
 */
function handleTicketCreated($data) {
    // Create ticket in helpdesk
    // Notify team
    // Create related issue
    return 'ticket_processed';
}

/**
 * Log webhook
 */
function logWebhook($event, $data, $results) {
    Capsule::table('mod_webhook_received')->insert([
        'event' => $event,
        'payload' => json_encode($data),
        'results' => json_encode($results),
        'created_at' => Carbon::now()->toDateTimeString()
    ]);
}
```

### Step 4: Retry Failed Webhooks

```php
<?php
/**
 * Retry failed webhook deliveries
 */
add_hook('DailyCronJob', 1, function($vars) {
    $failedWebhooks = Capsule::table('mod_webhook_logs')
        ->where('http_code', '>=', 400)
        ->where('created_at', '>=', Carbon::now()->subHours(24)->toDateTimeString())
        ->get();
    
    $retryCount = 0;
    
    foreach ($failedWebhooks as $webhook) {
        if ($webhook->retry_count >= 3) {
            continue; // Max retries reached
        }
        
        $payload = json_encode([
            'event' => $webhook->event,
            'timestamp' => $webhook->created_at,
            'data' => json_decode($webhook->data, true)
        ]);
        
        // Retry delivery
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => \WHMCS\Config\Setting::getValue('WebhookUrl'),
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-Webhook-Signature' => hash_hmac('sha256', $payload, $secret)
            ]
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        // Update retry count
        Capsule::table('mod_webhook_logs')
            ->where('id', $webhook->id)
            ->update([
                'retry_count' => ($webhook->retry_count ?? 0) + 1,
                'last_retry' => Carbon::now()->toDateTimeString()
            ]);
        
        if ($httpCode >= 200 && $httpCode < 300) {
            $retryCount++;
        }
    }
    
    logActivity("Webhook retry completed: $retryCount successful");
    
    return ['retry_count' => $retryCount];
});
```

## Best Practices
- Always verify webhook signatures
- Use HTTPS for webhooks
- Implement idempotent handlers
- Log all webhook events
- Implement retry logic
- Use queue for reliability
- Monitor webhook delivery
- Test webhooks thoroughly
- Handle timeouts gracefully
- Document webhook events
