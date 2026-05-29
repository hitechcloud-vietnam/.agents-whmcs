# WHMCS Webhook Testing Workflow

## Overview
This workflow provides comprehensive guidance for testing WHMCS webhooks, ensuring reliable event handling and delivery confirmation.

## Prerequisites
- WHMCS installation (v8.0+)
- Webhook endpoint configured
- Testing tools (ngrok, Postman, curl)
- Requestbin or similar for inspection

## Step-by-Step Guide

### Step 1: Configure Webhook Environment

#### Set Up Local Tunnel (for development)
```bash
# Install ngrok
# macOS
brew install ngrok

# Linux
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update && sudo apt install ngrok

# Windows (with Chocolatey)
choco install ngrok
```

#### Configure Webhook
```php
// modules/addons/yourmodule/webhook.php
<?php
// WHMCS Webhook Handler

use WHMCS\Module\Addon\YourModule\WebhookHandler;

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../vendor/autoload.php';

// Validate request
$handler = new WebhookHandler();

try {
    $payload = file_get_contents('php://input');
    $signature = $_SERVER['HTTP_X_WHMCS_SIGNATURE'] ?? '';
    
    // Verify signature
    if (!$handler->verifySignature($payload, $signature)) {
        http_response_code(403);
        die(json_encode(['error' => 'Invalid signature']));
    }
    
    // Parse payload
    $data = json_decode($payload, true);
    
    if (json_last_error() !== JSON_ERROR_NONE) {
        throw new Exception('Invalid JSON payload');
    }
    
    // Process webhook
    $handler->process($data);
    
    http_response_code(200);
    echo json_encode(['success' => true]);
    
} catch (Exception $e) {
    http_response_code(400);
    echo json_encode(['error' => $e->getMessage()]);
}
```

### Step 2: Write Webhook Handler
```php
// modules/addons/yourmodule/src/WebhookHandler.php
<?php
namespace WHMCS\Module\Addon\YourModule;

class WebhookHandler
{
    private string $secret;
    private array $validEvents = [
        'ClientCreate',
        'ClientUpdate',
        'ClientDelete',
        'OrderPlaced',
        'OrderPaid',
        'InvoiceCreated',
        'InvoicePaid',
        'ServiceCreate',
        'ServiceUpdate',
        'ServiceSuspend',
        'ServiceTerminate',
    ];

    public function __construct()
    {
        $this->secret = get_config('yourmodule_webhook_secret');
    }

    public function verifySignature(string $payload, string $signature): bool
    {
        $expected = 'sha256=' . hash_hmac('sha256', $payload, $this->secret);
        return hash_equals($expected, $signature);
    }

    public function process(array $data): void
    {
        $event = $data['event'] ?? '';
        
        if (!in_array($event, $this->validEvents)) {
            throw new Exception("Unknown event: $event");
        }
        
        $method = 'handle' . str_replace('_', '', ucwords($event, '_'));
        
        if (method_exists($this, $method)) {
            $this->$method($data);
        }
    }

    private function handleClientCreate(array $data): void
    {
        $clientId = $data['client_id'] ?? null;
        $email = $data['email'] ?? '';
        
        // Sync to external system
        $this->syncClient($clientId, $email);
    }

    private function handleOrderPaid(array $data): void
    {
        $orderId = $data['order_id'] ?? null;
        $invoiceId = $data['invoice_id'] ?? null;
        
        // Process order fulfillment
        $this->fulfillOrder($orderId, $invoiceId);
    }

    private function syncClient(int $clientId, string $email): void
    {
        logActivity("Webhook: Syncing client $clientId ($email)");
    }

    private function fulfillOrder(int $orderId, int $invoiceId): void
    {
        logActivity("Webhook: Fulfilling order $orderId (invoice: $invoiceId)");
    }
}
```

### Step 3: Test Webhooks

#### Test with curl
```bash
# Test webhook locally
curl -X POST http://localhost/modules/addons/yourmodule/webhook.php \
  -H "Content-Type: application/json" \
  -H "X-WHMCS-Signature: sha256=test" \
  -d '{
    "event": "ClientCreate",
    "client_id": 1,
    "email": "test@example.com",
    "firstname": "John",
    "lastname": "Doe"
  }'

# Test with valid signature
SECRET="your_webhook_secret"
PAYLOAD='{"event":"ClientCreate","client_id":1}'
SIGNATURE=$(echo -n "$PAYLOAD" | openssl dgst -sha256 -hmac "$SECRET" | cut -d' ' -f2)

curl -X POST http://localhost/modules/addons/yourmodule/webhook.php \
  -H "Content-Type: application/json" \
  -H "X-WHMCS-Signature: sha256=$SIGNATURE" \
  -d "$PAYLOAD"
```

#### Postman Collection
```json
{
  "info": {
    "name": "WHMCS Webhook Testing",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Client Create Webhook",
      "request": {
        "method": "POST",
        "header": [
          {
            "key": "Content-Type",
            "value": "application/json"
          },
          {
            "key": "X-WHMCS-Signature",
            "value": "sha256={{signature}}"
          }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n    \"event\": \"ClientCreate\",\n    \"client_id\": 1,\n    \"email\": \"test@example.com\"\n}"
        },
        "url": {
          "raw": "{{baseUrl}}/modules/addons/yourmodule/webhook.php",
          "host": ["{{baseUrl}}"],
          "path": ["modules", "addons", "yourmodule", "webhook.php"]
        }
      },
      "event": [
        {
          "listen": "test",
          "script": {
            "exec": [
              "pm.test('Webhook succeeds', function() {",
              "    pm.response.to.have.status(200);",
              "});",
              "",
              "pm.test('Response is JSON', function() {",
              "    pm.response.to.be.json;",
              "});"
            ]
          }
        }
      ]
    }
  ]
}
```

### Step 4: Automated Webhook Tests
```php
// tests/WebhookTest.php
<?php
namespace WHMCS\Tests;

use PHPUnit\Framework\TestCase;

class WebhookTest extends TestCase
{
    private string $webhookUrl;
    private string $secret;

    protected function setUp(): void
    {
        parent::setUp();
        $this->webhookUrl = 'http://localhost/modules/addons/yourmodule/webhook.php';
        $this->secret = 'test_secret';
    }

    public function testValidSignatureAccepted()
    {
        $payload = json_encode(['event' => 'ClientCreate', 'client_id' => 1]);
        $signature = 'sha256=' . hash_hmac('sha256', $payload, $this->secret);

        $ch = curl_init($this->webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-WHMCS-Signature: ' . $signature,
            ],
            CURLOPT_RETURNTRANSFER => true,
        ]);

        $response = curl_exec($ch);
        $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $this->assertEquals(200, $status);
        $this->assertStringContainsString('success', $response);
    }

    public function testInvalidSignatureRejected()
    {
        $payload = json_encode(['event' => 'ClientCreate', 'client_id' => 1]);

        $ch = curl_init($this->webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-WHMCS-Signature: sha256=invalid',
            ],
            CURLOPT_RETURNTRANSFER => true,
        ]);

        $response = curl_exec($ch);
        $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $this->assertEquals(403, $status);
    }

    public function testInvalidJsonRejected()
    {
        $ch = curl_init($this->webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => 'invalid json',
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
            CURLOPT_RETURNTRANSFER => true,
        ]);

        $response = curl_exec($ch);
        $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $this->assertEquals(400, $status);
    }

    public function testUnknownEventRejected()
    {
        $payload = json_encode(['event' => 'UnknownEvent', 'data' => 'test']);
        $signature = 'sha256=' . hash_hmac('sha256', $payload, $this->secret);

        $ch = curl_init($this->webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-WHMCS-Signature: ' . $signature,
            ],
            CURLOPT_RETURNTRANSFER => true,
        ]);

        $response = curl_exec($ch);
        $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $this->assertEquals(400, $status);
    }

    /**
     * @dataProvider eventProvider
     */
    public function testAllSupportedEvents(string $event)
    {
        $payload = json_encode(['event' => $event, 'client_id' => 1]);
        $signature = 'sha256=' . hash_hmac('sha256', $payload, $this->secret);

        $ch = curl_init($this->webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-WHMCS-Signature: ' . $signature,
            ],
            CURLOPT_RETURNTRANSFER => true,
        ]);

        $response = curl_exec($ch);
        $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $this->assertEquals(200, $status, "Event $event should be handled");
    }

    public function eventProvider(): array
    {
        return [
            ['ClientCreate'],
            ['ClientUpdate'],
            ['ClientDelete'],
            ['OrderPlaced'],
            ['OrderPaid'],
            ['InvoiceCreated'],
            ['InvoicePaid'],
        ];
    }
}
```

### Step 5: Test with ngrok
```bash
# Start ngrok tunnel
ngrok http 80

# Note the forward URL
# Forwarding  https://abc123.ngrok.io -> http://localhost

# Configure WHMCS webhook URL to the ngrok URL
# WHMCS Admin > Configuration > Module Settings > Webhook URL
```

### Step 6: Verify Webhook in WHMCS
```php
// WHMCS Admin > Configuration > System Settings > Module Settings
// Or via API
$whmcs = new WHMCSApi([
    'url' => 'https://your-whmcs.com/includes/api.php',
    'username' => 'admin_user',
    'password' => 'api_password_hash',
]);

// Test webhook
$whmcs->post('webhooks/test', [
    'webhook_url' => 'https://your-app.com/webhook',
    'event' => 'ClientCreate',
]);
```

### Step 7: Run Automated Tests
```bash
# Run webhook unit tests
./vendor/bin/phpunit tests/WebhookTest.php

# Run with verbose output
./vendor/bin/phpunit tests/WebhookTest.php --testdox

# Run specific test
./vendor/bin/phpunit tests/WebhookTest.php --filter testValidSignatureAccepted
```

## Webhook Testing Checklist

### Security
- [ ] Signature verification implemented
- [ ] Invalid signatures rejected
- [ ] Rate limiting configured
- [ ] IP whitelisting enabled
- [ ] HTTPS required

### Reliability
- [ ] Idempotent handlers
- [ ] Retry mechanism configured
- [ ] Dead letter queue configured
- [ ] Timeout handling
- [ ] Logging enabled

### Validation
- [ ] Event type validated
- [ ] Payload schema validated
- [ ] Required fields checked
- [ ] Data sanitized
- [ ] Error responses clear

## Common Webhook Issues

| Issue | Solution |
|-------|----------|
| Signature mismatch | Check secret key, encoding |
| Timeout errors | Increase timeout, implement async |
| Duplicate events | Use idempotency keys |
| Missing events | Check webhook configuration |
| Invalid payload | Validate JSON, check content-type |
